### **项目架构设计方案（第一部分）**  

# 项目架构设计方案  

## 一、项目概述  
### 1.1 项目名称  
### 1.2 核心目标  
### 1.3 适用场景  

## 二、架构设计原则  
### 2.1 遵循 WordPress & WooCommerce 规范  
### 2.2 模块化设计  
### 2.3 网络级与子站级隔离  
### 2.4 安全性  

## 三、模块划分与职责  
### 3.1 各模块职责概述  
### 3.2 APIClient（API 封装模块）  
### 3.3 PaymentUIModule（支付 UI 管理）  
### 3.4 NotificationModule（通知管理）  
### 3.5 OrderProcessor（订单处理）  
### 3.6 WebhookManager（Webhook 处理）  
### 3.7 AdminModule（后台管理）  

## 四、技术实现路径  
### 4.1 支付回调地址动态生成  
### 4.2 订单存储 `site_id`  
### 4.3 Webhook 自动注册与维护  
### 4.4 网络级与子站级管理界面  

## 五、优化分析与调整  
### 5.1 模块通信解耦（事件驱动）  
### 5.2 支付配置继承机制优化  

## 六、关键代码结构  
### 6.1 模块注册与加载（PluginManager）  
### 6.2 支付状态同步（APIClient）  
### 6.3 Webhook 安全验证（WebhookSecurity）  

## 七、部署与兼容性  
### 7.1 多站点兼容性（子站独立域名支持）  
### 7.2 数据库表设计  

## 八、未来扩展方向  
### 8.1 事件驱动架构增强  
### 8.2 支付网关抽象层  
### 8.3 自动化测试覆盖  

## 九、数据库与数据隔离设计  
### 9.1 数据存储策略（网络级 & 子站级）  
### 9.2 日志隔离设计（支付日志、Webhook 日志）  
### 9.3 分表查询优化  

## 十、安全性增强设计  
### 10.1 支付请求防篡改（签名机制）  
### 10.2 Webhook 安全防护（IP 白名单、重放攻击防御）  
### 10.3 敏感数据加密  

## 十一、网络级与子站级管理界面实现  
### 11.1 网络级管理界面（主站设置）  
### 11.2 子站级管理界面（独立配置）  
### 11.3 权限控制  

## 十二、异常处理与容灾机制  
### 12.1 支付流程异常处理（二维码过期、支付超时）  
### 12.2 Webhook 失败容灾（自动降级、人工干预通道）  
### 12.3 日志与监控（ELK 集成、关键指标监控）  

## 十三、性能优化方案  
### 13.1 静态资源缓存优化  
### 13.2 数据库查询优化（索引 & 分表）  
### 13.3 异步任务队列（WP_Background_Process）  

## 十四、测试与部署流程  
### 14.1 单元测试覆盖（PHPUnit）  
### 14.2 灰度发布策略（测试子站 -> 正式环境）  
### 14.3 回滚方案（旧版本回退机制）  

## 十五、最终架构总结  
### 15.1 主要架构优势  
### 15.2 待优化点（支付风控、国际化支持）  

---

#### **一、项目概述**  
**项目名称**：WordPress Multisite 微信 & 支付宝支付插件  
**核心目标**：  
1. 在 WordPress Multisite + WooCommerce 环境下，提供高效、安全的微信支付 & 支付宝支付功能。  
2. 支持网络级统一管理或子站独立管理，严格遵守 WordPress 和 WooCommerce 编码规范。  
3. 模块化设计，各功能解耦，支持单独启用/禁用。  

**适用场景**：  
- WordPress Multisite 网络，主站统一管理支付配置，子站继承规则。  
- 单站点独立激活时，子站管理员可独立配置支付参数。  

---

#### **二、架构设计原则**  
1. **遵循 WordPress & WooCommerce 规范**：  
   - 使用 `WP_Query`、`WC_Order` 等标准 API。  
   - 通过 WordPress Hook（Action/Filter）扩展功能，避免直接修改核心代码。  
2. **模块化设计**：  
   - 按功能划分独立模块（支付UI、Webhook、通知服务等），模块间通过事件驱动通信。  
   - 模块注册机制：通过 `PluginManager` 类动态加载模块。  
3. **网络级与子站级隔离**：  
   - 数据库表按 `site_id` 隔离（如日志、配置）。  
   - 网络级配置存储于 `wp_sitemeta`，子站配置存储于 `wp_options`。  
4. **安全性**：  
   - Webhook 请求需通过 HMAC-SHA256 签名验证。  
   - 支付回调 URL 动态生成，防止伪造请求。  

---

#### **三、模块划分与职责**  
| **模块名称**          | **职责**                                                                 | **依赖关系**              |  
|-----------------------|--------------------------------------------------------------------------|---------------------------|  
| **PaymentUIModule**   | 管理前端支付体验（倒计时、二维码刷新、自动跳转）                         | 无                        |  
| **NotificationModule** | 处理短信 & 邮件通知，支持网络级配置继承                                  | 依赖 `WC_Email` 类        |  
| **OrderProcessor**    | 处理订单支付 & 退款状态同步，存储 `site_id`，防重复退款                  | 依赖 `WC_Order`           |  
| **WebhookManager**    | Webhook 自动注册、失败重试、安全验证、日志记录                           | 依赖 `WC_Webhook`         |  
| **AdminModule**       | 提供网络级 & 子站级管理界面（支付配置、日志查看、统计）                  | 依赖 WordPress Admin API  |  
| **APIClient**         | **封装微信 & 支付宝 API 调用，必须使用官方 SDK 进行集成**                 | 无                        |  

**SDK 集成要求：**  
- **微信支付：** 使用 `wechatpay-php`（https://github.com/wechatpay-apiv3/wechatpay-php）  
- **支付宝支付：** 使用 `alipay-sdk-php`（https://github.com/alipay/alipay-sdk-php-all）  

---

#### **四、技术实现路径**  

##### **1. 支付回调地址动态生成**  
**问题**：多站点环境下，子站需独立回调地址以匹配订单。  
**实现方案**：  
- **主站回调地址**：`https://tao.ooo/wc-api/wechat_callback`  
- **子站回调地址**：`https://sub.tao.ooo/wc-api/wechat_callback`  
- **动态生成逻辑**：  
  ```php  
  function generate_callback_url($type) {  
      $blog_id = get_current_blog_id();  
      $base_url = get_site_url($blog_id);  
      return $base_url . "/wc-api/{$type}_callback";  
  }  
  ```  

##### **2. 订单存储 `site_id`**  
**数据库设计**：  
- 扩展 `wp_posts` 表，通过 `_site_id` Meta 字段标记订单所属子站。  
- **Hook 实现**：  
  ```php  
  add_action('woocommerce_checkout_create_order', function ($order) {  
      $order->update_meta_data('_site_id', get_current_blog_id());  
  });  
  ```  

##### **3. Webhook 自动注册与维护**  
**实现步骤**：  
1. **插件激活时**：遍历所有子站，为每个站点创建独立 Webhook。  
2. **子站创建时**：通过 `wpmu_new_blog` Hook 自动注册 Webhook。  
3. **Webhook 参数**：  
   ```php  
   $webhook = new WC_Webhook();  
   $webhook->set_name('微信支付回调');  
   $webhook->set_topic('payment.wechat');  
   $webhook->set_delivery_url(generate_callback_url('wechat'));  
   $webhook->set_secret(wp_generate_password(32));  
   ```  

##### **4. 网络级与子站级管理地址**  
- **网络级管理界面**：  
  URL: `https://tao.ooo/wp-admin/network/admin.php?page=wc-multisite-payment`  
  功能：配置全局支付网关、强制启用短信通知、查看全站支付日志。  
- **子站级管理界面**：  
  URL: `https://sub.tao.ooo/wp-admin/admin.php?page=wc-payment-settings`  
  功能：子站独立配置支付 API、查看本地日志、手动同步订单状态。  

---

#### **五、优化分析与调整**  

##### **优化点 1：模块通信解耦**  
**问题**：原方案中 `WebhookManager` 直接调用 `OrderProcessor` 的方法，导致耦合。  
**优化方案**：  
- 使用 WordPress 的 `do_action` 实现事件驱动：  
  ```php  
  // Webhook 接收到支付成功事件时触发  
  do_action('wc_payment_success', $order_id, $transaction_id);  
  // OrderProcessor 监听该事件  
  add_action('wc_payment_success', [$this, 'update_order_status']);  
  ```  

---

#### **六、部署与兼容性**  
- **多站点支持**：确保 `APIClient` 模块在不同子站独立运行，不影响其他站点的支付逻辑。  
- **数据库索引优化**：确保 `_site_id` 字段添加索引，提高查询性能。  
  ```sql  
  ALTER TABLE wp_postmeta ADD INDEX meta_key_site_id (meta_key, meta_value(20));  
  ```  

---

### **总结**  
本次更新对 `APIClient` 模块进行了优化，确保所有支付集成功能必须使用官方 SDK，并提升了数据库查询性能和模块解耦性。

