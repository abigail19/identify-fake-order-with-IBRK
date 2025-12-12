# identify-fake-order-with-IBRK
# identify-fake-order-with-IBRK
pip install ib_insync
from ib_insync import IB, Order, Contract, OrderStatus
import pandas as pd
from datetime import datetime
import time
from typing import List, Dict

# 全局变量：存储订单数据
order_records: List[Dict] = []

class IBOrderDataCollector:
    def __init__(self, host: str = '127.0.0.1', port: int = 7497, client_id: int = 1):
        """初始化IB连接"""
        self.ib = IB()
        self.host = host
        self.port = port
        self.client_id = client_id
        self.order_map = {}  # 缓存订单信息：orderId -> 订单详情

    def connect(self) -> bool:
        """连接IB TWS/IB Gateway"""
        try:
            self.ib.connect(self.host, self.port, clientId=self.client_id)
            print(f"成功连接IB API：{self.host}:{self.port}")
            # 注册订单状态监听回调
            self.ib.orderStatusEvent += self.on_order_status
            self.ib.openOrderEvent += self.on_open_order
            self.ib.cancelOrderEvent += self.on_cancel_order
            return True
        except Exception as e:
            print(f"连接失败：{e}")
            return False

    def on_open_order(self, trade, order):
        """监听新订单创建事件"""
        create_time = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
        self.order_map[order.orderId] = {
            'order_id': order.orderId,
            'create_time': create_time,
            'cancel_time': '',
            'status': self._get_order_status_desc(order.status),
            'contract': f"{order.contract.symbol}-{order.contract.secType}-{order.contract.exchange}"
        }
        print(f"新订单创建：ID={order.orderId} | 时间={create_time} | 状态={order.status}")

    def on_order_status(self, trade, orderStatus):
        """监听订单状态更新事件"""
        if orderStatus.orderId not in self.order_map:
            return
        # 更新订单状态
        self.order_map[orderStatus.orderId]['status'] = self._get_order_status_desc(orderStatus.status)
        # 如果是撤单状态，记录撤单时间
        if orderStatus.status in [OrderStatus.Cancelled, OrderStatus.Canceled]:
            cancel_time = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
            self.order_map[orderStatus.orderId]['cancel_time'] = cancel_time
            print(f"订单撤单：ID={orderStatus.orderId} | 撤单时间={cancel_time}")

    def on_cancel_order(self, order):
        """监听主动撤单事件（兜底）"""
        if order.orderId in self.order_map:
            cancel_time = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
            self.order_map[order.orderId]['cancel_time'] = cancel_time
            self.order_map[order.orderId]['status'] = '已撤单'
            print(f"主动撤单：ID={order.orderId} | 撤单时间={cancel_time}")

    def _get_order_status_desc(self, status: str) -> str:
        """将IB订单状态转换为中文描述"""
        status_mapping = {
            OrderStatus.Submitted: '已提交',
            OrderStatus.Filled: '已成交',
            OrderStatus.Cancelled: '已撤单',
            OrderStatus.Canceled: '已撤单',  # 兼容不同拼写
            OrderStatus.Inactive: '未激活',
            OrderStatus.PendingCancel: '撤单中',
            OrderStatus.PendingSubmit: '提交中'
        }
        return status_mapping.get(status, status)

    def get_level2_data(self, contract: Contract, duration: int = 60):
        """抓取Level 2深度数据（买卖盘）"""
        print(f"开始抓取{contract.symbol}的Level 2数据，持续{duration}秒...")
        # 订阅Level 2市场深度数据
        ticker = self.ib.reqMktDepth(contract, numRows=10)  # numRows=10表示获取10档深度
        time.sleep(duration)
        # 取消订阅
        self.ib.cancelMktDepth(contract)
        return ticker

    def export_to_csv(self, csv_path: str = 'order_data.csv'):
        """将订单数据导出为指定格式的CSV"""
        # 转换订单映射为列表，过滤无效数据
        global order_records
        order_records = [v for k, v in self.order_map.items() if v['order_id']]
        
        # 构建DataFrame
        df = pd.DataFrame(order_records)[['order_id', 'create_time', 'cancel_time', 'status']]
        # 处理空的撤单时间（已成交/未撤单订单）
        df['cancel_time'] = df['cancel_time'].replace('', None)
        # 保存CSV
        df.to_csv(csv_path, index=False, encoding='utf-8')
        print(f"订单数据已导出至：{csv_path}")
        print(f"共导出 {len(df)} 条订单记录")
        return df

    def disconnect(self):
        """断开IB连接"""
        if self.ib.isConnected():
            self.ib.disconnect()
            print("已断开IB API连接")

def main():
    # 1. 初始化采集器
    collector = IBOrderDataCollector(port=7497, client_id=1)
    
    # 2. 连接IB TWS/IB Gateway
    if not collector.connect():
        return
    
    try:
        # 3. 定义要监控的合约（示例：丰田美股）
        contract = Contract()
        contract.symbol = 'TM'
        contract.secType = 'STK'
        contract.exchange = 'SMART'
        contract.currency = 'USD'
        
        # 4. 抓取Level 2数据（持续60秒，可根据需求调整）
        collector.get_level2_data(contract, duration=60)
        
        # 5. 等待订单状态稳定（可选）
        time.sleep(5)
        
        # 6. 导出为指定CSV格式
        collector.export_to_csv('order_data.csv')
        
    except Exception as e:
        print(f"执行出错：{e}")
    finally:
        # 7. 断开连接
        collector.disconnect()

if __name__ == "__main__":
    main()

#整理level2数据
import pandas as pd
from datetime import datetime, timedelta

def analyze_valid_orders(order_data_path, fake_threshold=5, stat_window=30):
    """
    剔除5分钟内撤单的虚假订单，统计剩余订单30分钟内的Bid/成交数据
    :param order_data_path: 订单数据文件路径（CSV）
    :param fake_threshold: 虚假订单撤单阈值（分钟），默认5分钟
    :param stat_window: 统计时间窗口（分钟），默认30分钟
    :return: 统计结果字典
    """
    # 1. 读取订单数据（字段：order_id, create_time, cancel_time, status, type）
    # type字段补充：Bid(挂单)/Trade(成交)，status：已成交/已撤单/待成交
    df = pd.read_csv(order_data_path)
    
    # 2. 数据预处理：时间格式转换
    try:
        df['create_time'] = pd.to_datetime(df['create_time'])
        df['cancel_time'] = pd.to_datetime(df['cancel_time'], errors='coerce')  # 未撤单则为NaT
    except Exception as e:
        raise ValueError(f"时间格式转换失败：{e}（请确保格式为YYYY-MM-DD HH:MM:SS）")
    
    # 3. 识别并删除5分钟内撤单的虚假订单
    # 筛选已撤单订单
    cancel_orders = df[df['status'] == '已撤单'].copy()
    if not cancel_orders.empty:
        # 计算撤单时间差（分钟）
        cancel_orders['time_diff_min'] = (cancel_orders['cancel_time'] - cancel_orders['create_time']).dt.total_seconds() / 60
        # 标记虚假订单（5分钟内撤单）
        fake_order_ids = cancel_orders[cancel_orders['time_diff_min'] <= fake_threshold]['order_id'].tolist()
        # 删除虚假订单
        valid_orders = df[~df['order_id'].isin(fake_order_ids)].copy()
        print(f"已删除{len(fake_order_ids)}笔5分钟内撤单的虚假订单")
    else:
        valid_orders = df.copy()
        print("无5分钟内撤单的虚假订单")
    
    # 4. 统计30分钟内的Bid/成交数据
    # 计算订单创建后30分钟的截止时间
    valid_orders['stat_end_time'] = valid_orders['create_time'] + timedelta(minutes=stat_window)
    # 当前时间（若需基于订单最新状态，可替换为df['update_time']）
    current_time = pd.to_datetime(datetime.now())
    
    # 筛选30分钟内的订单（创建时间在统计窗口内）
    window_orders = valid_orders[
        (valid_orders['create_time'] >= (current_time - timedelta(minutes=stat_window)))
    ].copy()
    
    # 统计核心指标
    # 30分钟内总有效订单数
    total_valid = len(window_orders)
    # 30分钟内Bid(挂单)订单数
    bid_orders = window_orders[window_orders['type'] == 'Bid']
    bid_count = len(bid_orders)
    # 30分钟内成交订单数
    trade_orders = window_orders[window_orders['status'] == '已成交']
    trade_count = len(trade_orders)
    # 30分钟内挂单成交率
    bid_trade_rate = trade_count / bid_count if bid_count > 0 else 0.0
    # 30分钟内撤单订单数（非虚假撤单）
    non_fake_cancel = window_orders[
        (window_orders['status'] == '已撤单') & 
        ~window_orders['order_id'].isin(fake_order_ids if 'fake_order_ids' in locals() else [])
    ]
    non_fake_cancel_count = len(non_fake_cancel)
    
    # 5. 构建统计结果
    stat_result = {
        "统计窗口（分钟）": stat_window,
        "30分钟内总有效订单数": total_valid,
        "30分钟内Bid(挂单)订单数": bid_count,
        "30分钟内成交订单数": trade_count,
        "30分钟内挂单成交率": round(bid_trade_rate * 100, 2),
        "30分钟内非虚假撤单订单数": non_fake_cancel_count,
        "30分钟内待成交订单数": len(window_orders[window_orders['status'] == '待成交'])
    }
    
    # 输出统计结果
    print("\n===== 30分钟内Bid/成交数据统计 =====")
    for key, value in stat_result.items():
        print(f"{key}：{value}")
    
    # 可选：保存30分钟内有效订单数据到CSV
    window_orders.to_csv('30min_valid_orders.csv', index=False, encoding='utf-8')
    print("\n30分钟内有效订单数据已保存至：30min_valid_orders.csv")
    
    return stat_result

# 测试调用
if __name__ == "__main__":
    try:
        # 替换为你的订单数据文件路径
        result = analyze_valid_orders("order_data.csv")
    except FileNotFoundError:
        print("错误：未找到订单数据文件，请检查路径是否正确")
    except Exception as e:
        print(f"程序执行出错：{e}")
