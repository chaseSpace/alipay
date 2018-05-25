写在前面：
    讲真，一开始接到这个任务我是拒绝的。因为支付宝官方没有提供Python的SDK环境，只有JAVA/PHP/.NET三种语言的SDK，这意味着签名&验签、HTTP接口请求等操作全都要自己手动实现，就算支付宝提供了签名、验签的算法说明，但仅靠它的文字描述就写出一个符合支付宝想法的算法很明显“任重道远”，我当然不会去尝试这条路。

    幸运的是，github总能给我惊喜，搜索“alipay python sdk”结果第一个就是，两百多颗星，这意味着接下来的路平坦许多。

正文：
     Github上的Alipay Python Sdk
   我接到的需求是做二维码支付（不同于支付宝官方的当面付/APP支付/手机网站支付/电脑网站/支付/等），流程如下：

   接收商品名称信息、订单金额、订单号 》》》请求“支付宝预付订单创建接口”根据返回的URL生成二维码》》》 用户扫码支付》》》一定时间内轮询订单状态，若超时未支付则关闭订单。

    操作： 
pip install python-alipay-sdk --upgrade

from alipay import AliPay
import time,qrcode
alipay_public_key_string = '''-----BEGIN PUBLIC KEY-----
    YOUR_ALIPAY_PUBLIC_KEY
-----END PUBLIC KEY-----'''
app_private_key_string = '''-----BEGIN RSA PRIVATE KEY-----
    YOUR_APP_PRIVATE_KEY
-----END RSA PRIVATE KEY-----'''
#注意：一个是支付宝公钥，一个是应用私钥
APP_ID = '2018052160210122'
NOTIFY_URL = "https://your_domain/alipay_callback"

def init_alipay_cfg():
    '''
    初始化alipay配置
    :return: alipay 对象
    '''
    alipay = AliPay(
        appid=APP_ID,
        app_notify_url=NOTIFY_URL,  # 默认回调url
        app_private_key_string=app_private_key_string,
        alipay_public_key_string=alipay_public_key_string,  # 支付宝的公钥，验证支付宝回传消息使用，不是你自己的公钥,
        sign_type="RSA2",  # RSA 或者 RSA2
        debug = False  # 默认False ,若开启则使用沙盒环境的支付宝公钥
    )
    return alipay

def get_qr_code(code_url):
    '''
    生成二维码
    :return None
    '''
    #print(code_url)
    qr = qrcode.QRCode(
        version=1,
        error_correction=qrcode.constants.ERROR_CORRECT_H,
        box_size=10,
        border=1
    )
    qr.add_data(code_url)  # 二维码所含信息
    img = qr.make_image()  # 生成二维码图片
    img.save(r'C:\Users\SEEMORE\Desktop\qr_test_ali.png')
    print('二维码保存成功！')

def preCreateOrder(subject:'order_desc' , out_trade_no:int, total_amount:(float,'eg:0.01')):
    '''
    创建预付订单
    :return None：表示预付订单创建失败  [或]  code_url：二维码url
    '''
    result = init_alipay_cfg().api_alipay_trade_precreate(
        subject=subject,
        out_trade_no=out_trade_no,
        total_amount=total_amount)
    print('返回值：',result)
    code_url = result.get('qr_code')
    if not code_url:
        print(result.get('预付订单创建失败：','msg'))
        return
    else:
        get_qr_code(code_url)
        #return code_url


def query_order(out_trade_no:int, cancel_time:int and 'secs'):
    '''
    :param out_trade_no: 商户订单号
    :return: None
    '''
    print('预付订单已创建,请在%s秒内扫码支付,过期订单将被取消！'% cancel_time)
    # check order status
    _time = 0
    for i in range(10):
        # check every 3s, and 10 times in all

        print("now sleep 2s")
        time.sleep(2)

        result = init_alipay_cfg().api_alipay_trade_query(out_trade_no=out_trade_no)
        if result.get("trade_status", "") == "TRADE_SUCCESS":
            print('订单已支付!')
            print('订单查询返回值：',result)
            break

        _time += 2
        if _time >= cancel_time:
            cancel_order(out_trade_no,cancel_time)
            return

def cancel_order(out_trade_no:int, cancel_time=None):
    '''
    撤销订单
    :param out_trade_no:
    :param cancel_time: 撤销前的等待时间(若未支付)，撤销后在商家中心-交易下的交易状态显示为"关闭"
    :return:
    '''
    result = init_alipay_cfg().api_alipay_trade_cancel(out_trade_no=out_trade_no)
    #print('取消订单返回值：', result)
    resp_state = result.get('msg')
    action = result.get('action')
    if resp_state=='Success':
        if action=='close':
            if cancel_time:
                print("%s秒内未支付订单，订单已被取消！" % cancel_time)
        elif action=='refund':
            print('该笔交易目前状态为：',action)

        return action

    else:
        print('请求失败：',resp_state)
        return


def need_refund(out_trade_no:str or int, refund_amount:int or float, out_request_no:str):
    '''
    退款操作
    :param out_trade_no: 商户订单号
    :param refund_amount: 退款金额，小于等于订单金额
    :param out_request_no: 商户自定义参数，用来标识该次退款请求的唯一性,可使用 out_trade_no_退款金额*100 的构造方式
    :return:
    '''
    result = init_alipay_cfg().api_alipay_trade_refund(out_trade_no=out_trade_no,
                                                       refund_amount=refund_amount,
                                                       out_request_no=out_request_no)

    if result["code"] == "10000":
        return result  #接口调用成功则返回result
    else:
        return result["msg"] #接口调用失败则返回原因

def refund_query(out_request_no:str, out_trade_no:str or int):
    '''
    退款查询：同一笔交易可能有多次退款操作（每次退一部分）
    :param out_request_no: 商户自定义的单次退款请求标识符
    :param out_trade_no: 商户订单号
    :return:
    '''
    result = init_alipay_cfg().api_alipay_trade_fastpay_refund_query(out_request_no, out_trade_no=out_trade_no)

    if result["code"] == "10000":
        return result  #接口调用成功则返回result
    else:
        return result["msg"] #接口调用失败则返回原因


if __name__ == '__main__':
    cancel_order(1527212120)
    subject = "话费余额充值"
    out_trade_no =int(time.time())
    total_amount = 0.01
    preCreateOrder(subject,out_trade_no,total_amount)

    query_order(out_trade_no,40)

    print('5s后订单自动退款')
    time.sleep(5)
    print(need_refund(out_trade_no,0.01,111))

    print('5s后查询退款')
    time.sleep(5)
    print(refund_query(out_request_no=111, out_trade_no=out_trade_no))
    #操作完登录 https://authsu18.alipay.com/login/index.htm中的对账中心查看是否有一笔交易生成并退款

'''
官文：https://docs.open.alipay.com/api_1/
创建预付订单返回值：
{'code': '10000', 'msg': 'Success', 'out_trade_no': '1527214200', 'qr_code': 'https://qr.alipay.com/bax09560qqw1epttm5i8006e'}

取消订单返回值示例：
{'code': '10000', 'msg': 'Success', 'action': 'close'（交易已取消）, 'out_trade_no': '1527212120', 'retry_flag': 'N',
 'trade_no': '2018052521001004480221282310'}

订单查询返回值：
{'code': '10000', 'msg': 'Success', 'buyer_logon_id': 'cha***@icloud.com', 'buyer_pay_amount': '0.01',
 'buyer_user_id': '2088012915825485', 'fund_bill_list': [{'amount': '0.01', 'fund_channel': 'ALIPAYACCOUNT'}],
  'invoice_amount': '0.01', 'out_trade_no': '1527217434', 'point_amount': '0.00', 'receipt_amount': '0.01',
   'send_pay_date': '2018-05-25 11:04:08', 'total_amount': '0.01', 'trade_no': '2018052521001004480221487538',
    'trade_status': 'TRADE_SUCCESS'}

退款返回值示例：
{'code': '10000', 'msg': 'Success', 'buyer_logon_id': 'cha***@icloud.com', 'buyer_user_id': '2088012915825485', 'fund_change': 'Y',
 'gmt_refund_pay': '2018-05-25 10:08:05', 'out_trade_no': '1527211312', 'refund_detail_item_list': [{'amount': '0.01', 'fund_channel': 'ALIPAYACCOUNT'}],
  'refund_fee': '0.01', 'send_back_fee': '0.01', 'trade_no': '2018052521001004480221209563'}
  
退款查询返回值示例：
{'code': '10000', 'msg': 'Success', 'out_request_no': '111', 'out_trade_no': '1527216203', 'refund_amount': '0.01',
 'total_amount': '0.01', 'trade_no': '2018052521001004480221450113'}
'''
-有用可以点个赞-
