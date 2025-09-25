### odoo18 多work 下也是好使的
#### 飞书 自动监听 订阅的事件。 不同的事件掉不同的方法，根据 odoo的继承机制，可以实现，这里调用，其他继承飞书模块的模块去集中处理业务

#### get_all_approve_code 这方法返回被订阅的审批、其他事件的instance_code 
#### approval_instance_task 审批任务 事件 
#### approval_instance_event 审批实例  事件


```python

cache = TTLCache(maxsize=5, ttl=60 * 25)
class FeishuListenerThread(threading.Thread):
    def __init__(self, registry, interval=10):
        super().__init__(daemon=True)  # daemon=True 确保主进程退出线程自动结束
        self.registry = registry
        self.interval = interval
        self._stop_flag = False

    def run(self):
        _logger.info("Feishu Listener thread started")
        while not self._stop_flag:
            try:
                # 使用 ORM 调用数据库
                # TODO: 替换成你的逻辑，比如监听接口或轮询
                self.create_agent_token( )
                _logger.info("Total partners: %s", 10)
            except Exception as e:
                _logger.exception("Feishu Listener error: %s", e)
            time.sleep(self.interval)

    def stop(self):
        self._stop_flag = True

    def create_agent_token(self):
        with self.registry.cursor() as cr_new:
            # redis 生产者消费者模式中的消费者部分的代码
            env = Environment(cr_new, SUPERUSER_ID, {})
            icp = env['ir.config_parameter'].sudo()
            app_id = icp.get_param('kinlim.feishu.app.id')
            app_secret = icp.get_param('kinlim.feishu.app.secret')
            encrypt_key = icp.get_param('kinlim.feishu.encrypt.key')
            verification_token = icp.get_param('kinlim.feishu.verification.token')
            client = lark.Client.builder() \
                .app_id(app_id) \
                .app_secret(app_secret) \
                .log_level(lark.LogLevel.INFO) \
                .build()
            # 构造请求对象
            approve_codes = env['feishu.contact'].get_all_approve_code()
            for approve_code in approve_codes:
                request: SubscribeApprovalRequest = SubscribeApprovalRequest.builder() \
                    .approval_code(approve_code).build()
                # 发起请求
                response: SubscribeApprovalResponse = client.approval.v4.approval.subscribe(request)
                _logger.info(response)
            def  make_approval_task_handler(event):
                with self.registry.cursor() as cr:
                    env_inner = odoo.api.Environment(cr, odoo.SUPERUSER_ID, {})
                    _logger.info(event.event)
                    env_inner['feishu.contact'].sudo().approval_instance_task(event)
            def make_approval_instance_handler(event):
                with self.registry.cursor() as cr:
                    env_inner = odoo.api.Environment(cr, odoo.SUPERUSER_ID, {})
                    _logger.info(f'{event.event}----------1233')
                    env_inner['feishu.contact'].sudo().approval_instance_event(event)
            event_handler = lark.EventDispatcherHandler.builder(encrypt_key, verification_token) \
                .register_p1_customized_event("approval_instance", make_approval_instance_handler) \
                .build()
            cli = lark.ws.Client(app_id, app_secret,
                                 event_handler=event_handler, log_level=lark.LogLevel.INFO)
            cli.start()

_thread = None
def start_listener():
    global _thread
    if _thread is None:
        registry = Registry(odoo.tools.config['db_name'])
        _logger.info(odoo.tools.config['db_name'])
        _thread = FeishuListenerThread(registry, interval=10)
        _thread.start()
        _logger.info("✅ Feishu Listener thread started in main process")
    else:
        _logger.info("⚠️ Feishu Listener already running, skip duplicate start")
# 关键：server_wide_modules 会在主进程加载模块时执行 __init__.py
start_listener()
```
