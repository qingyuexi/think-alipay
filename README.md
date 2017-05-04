# think-alipay
The ThinkPHP5 Alipay Package

## 安装
> composer require qingyuexi/think-alipay

## 配置
### 支付参数配置
```
$config = array(
    // 即时到账方式
    'payment_type' => '1',
    // 传输协议
    'transport' => 'http',
    // 编码方式
    'input_charset' => strtolower('utf-8'),
    // 签名方法
    'sign_type' => strtoupper('MD5'),
    // 支付完成异步通知调用地址
    'notify_url' => $this->appUrl . url('api/pay/alipayNotifyUrl'),
    // 支付完成同步返回地址
    'return_url' => $this->appUrl . url('api/pay/alipayReturnUrl', array("id" => input('param.id'))),
    // 证书路径
    'cacert' => DATA_PATH . 'cacert.pem',
    // 支付宝商家 ID
    'partner' => $alipay['config']['partner'],
    // 支付宝商家 KEY
    'key' => $alipay['config']['key'],
    // 支付宝商家注册邮箱
    'seller_email' => $alipay['config']['account']
);
```
在ThinkPHP5中手动导入
```
<?php
namespace app\api\controller;
use app\api\controller\BaseController;

class PayController extends BaseController
{
	public $appUrl = "";
    public function _initialize()
    {
        if (request()->isOptions()){
            abort(json(true,200));
        }
        $this->appUrl = request()->root(true);
    }
    /**
     * @return array
     */
    public function alipayInit()
    {
        $alipay = model("Payment")->where('type','alipay')->find();
        $config = array(
            // 即时到账方式
            'payment_type' => '1',
            // 传输协议
            'transport' => 'http',
            // 编码方式
            'input_charset' => strtolower('utf-8'),
            // 签名方法
            'sign_type' => strtoupper('MD5'),
            // 支付完成异步通知调用地址
            'notify_url' => $this->appUrl . url('api/pay/alipayNotifyUrl'),
            // 支付完成同步返回地址
            'return_url' => $this->appUrl . url('api/pay/alipayReturnUrl', array("id" => input('param.id'))),
            // 证书路径
            'cacert' => DATA_PATH . 'cacert.pem',
            // 支付宝商家 ID
            'partner' => $alipay['config']['partner'],
            // 支付宝商家 KEY
            'key' => $alipay['config']['key'],
            // 支付宝商家注册邮箱
            'seller_email' => $alipay['config']['account']
        );
        return $config;
    }
    //支付宝支付
    public function alipay()
    {
        $id = input('param.id');
        $order = model('Order')->with('user')->find($id);

        Vendor("qingyuexi.Alipay.Alipay");

        $out_trade_no = $order["orderid"];
        $subject = $order["orderid"];
        $total_fee = floatval($order["totalprice"]);
        $body = $order["orderid"];
        $show_url = $this->appUrl;
        $anti_phishing_key = "";
        $exter_invoke_ip = "";

        /************************************************************/
        $is_mobile = request()->isMobile() ? true :false;
        
        $alipay_config = $this->alipayInit();
        $alipay = new \Alipay($alipay_config, $is_mobile);

        if ($is_mobile) {
            $params = $alipay->prepareMobileTradeData(array(
                'out_trade_no' => $out_trade_no,
                'subject' => $subject,
                'body' => $body,
                'total_fee' => $total_fee,
                'merchant_url' => $this->appUrl . "/api/index/index#/order/" . $id,
                'req_id' => date('Ymdhis-')
            ));
            $url = $alipay->buildRequestUrl($params);
            $data['url'] = $url;
            return json(['data' => $data, 'msg' => '支付链接', 'code' => 1]);
        } else {
            $html = $alipay->buildRequestFormHTML(array(
                 "service" => "create_direct_pay_by_user",
                 "partner" => trim($alipay_config['partner']),
                 "payment_type" => $alipay_config['payment_type'],
                 "notify_url" => $alipay_config['notify_url'],
                 "return_url" => $alipay_config['return_url'],
                 "seller_id" => $alipay_config['partner'],
                 "out_trade_no" => $out_trade_no,
                 "subject" => $subject,
                 "total_fee" => $total_fee,
                 "body" => $body,
                 "show_url" => $show_url,
                 "anti_phishing_key" => $anti_phishing_key,
                 "exter_invoke_ip" => $exter_invoke_ip,
                 "_input_charset" => trim(strtolower($alipay_config['input_charset']))
             ));
             echo $html;
        }
    }  
    /**
     *回调地址
     */
    public function alipayNotifyUrl()
    {
        
        Vendor("qingyuexi.Alipay.Alipay");
        if(isset($_POST['notify_data'])){
            //手机支付
            $is_mobile = true;
        }else{
            //网页支付
            $is_mobile = false;
        }
        $alipay_config = $this->alipayInit();
        $alipay = new \Alipay($alipay_config, $is_mobile);
        $verify_result = $alipay->verifyCallback();

        if ($verify_result) {//验证成功
            /////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
            //请在这里加上商户的业务逻辑程序代
            //——请根据您的业务逻辑来编写程序（以下代码仅作参考）——

            //获取支付宝的通知返回参数，可参考技术文档中服务器异步通知参数列表
            if(isset($_POST['notify_data'])){
                $_POST = simplest_xml_to_array($_POST['notify_data']);
            }
            //商户订单号
            $out_trade_no = $_POST['out_trade_no'];
            //支付宝交易号
            $trade_no = $_POST['trade_no'];

            //交易状态
            $trade_status = $_POST['trade_status'];


            if ($_POST['trade_status'] == 'TRADE_FINISHED') {
                //判断该笔订单是否在商户网站中已经做过处理
                //如果没有做过处理，根据订单号（out_trade_no）在商户网站的订单系统中查到该笔订单的详细，并执行商户的业务程序
                //如果有做过处理，不执行商户的业务程序

                //注意：
                //该种交易状态只在两种情况下出现
                //1、开通了普通即时到账，买家付款成功后。
                //2、开通了高级即时到账，从该笔交易成功时间算起，过了签约时的可退款时限（如：三个月以内可退款、一年以内可退款等）后。

                //调试用，写文本函数记录程序运行情况是否正常
                //logResult("这里写入想要调试的代码变量值，或其他运行的结果记录");
            } else if ($_POST['trade_status'] == 'TRADE_SUCCESS') {
                //判断该笔订单是否在商户网站中已经做过处理
                //如果没有做过处理，根据订单号（out_trade_no）在商户网站的订单系统中查到该笔订单的详细，并执行商户的业务程序
                //如果有做过处理，不执行商户的业务程序

                //注意：
                //该种交易状态只在一种情况下出现——开通了高级即时到账，买家付款成功后。

                //调试用，写文本函数记录程序运行情况是否正常
                //logResult("这里写入想要调试的代码变量值，或其他运行的结果记录");

                $this->payTrue($out_trade_no, "", "支付宝支付");
            }

            //——请根据您的业务逻辑来编写程序（以上代码仅作参考）——

            echo "success";        //请不要修改或删除

            /////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
        } else {
            //验证失败
            echo "fail";

            //调试用，写文本函数记录程序运行情况是否正常
            //logResult("这里写入想要调试的代码变量值，或其他运行的结果记录");
        }
    }
}
/**
 * 最简单的XML转数组
 * @param string $xmlstring XML字符串
 * @return array XML数组
 */
function simplest_xml_to_array($xmlstring)
{
    return json_decode(json_encode((array)simplexml_load_string($xmlstring)), true);
}

```


