[TOC]

# Nhu cầu
- Kiểm tra connect của service đến các đối tượng khác liên quan (kafka, db, service khác,...)
- Có thể cấu hình để alert cảnh báo khi connect có vấn đề
- Có thể config ra grafana để theo dõi quan sát
- Có thể custom để check thêm 1 số logic nghiệp vụ nếu cần:
   - Số lượng data bản ghi trên service có vấn đề (rỗng, bất thường...)
---

# Áp dụng vào Code

## Thư viện cần cài đặt

- Nếu chỉ cần check connect thì có thể sử dụng các thư viện có sẵn:   
(dựa vào version .NET để cài các bản mới hơn, đây đang ở .NET 6)
```html
		<!--Kafka-->
		<PackageReference Include="AspNetCore.HealthChecks.Kafka" Version="8.0.1" />
		<!--Oracle-->
		<PackageReference Include="AspNetCore.HealthChecks.Oracle" Version="8.0.1" />
		<!--Uri-->
		<PackageReference Include="AspNetCore.HealthChecks.Uris" Version="8.0.1" />
		<!--xpose health status theo Prometheus format để Prometheus scrape (ví dụ cho Grafana dashboard).-->
		<PackageReference Include="AspNetCore.HealthChecks.Prometheus.Metrics" Version="8.0.1" />
		<!--Dùng để tạo UI healthcheck-->
		<PackageReference Include="AspNetCore.HealthChecks.UI.Client" Version="8.0.1" />
```

- Nếu cần custom thì cài đặt thêm
```html
<PackageReference Include="Microsoft.AspNetCore.Diagnostics.HealthChecks" Version="8.0.0" />
```
---

## Đăng ký các healcheck
### Tạo các class map từ appsettings
- Url:
    - Là các endpoint để gọi đến healthcheck
    - Ngoài các endpoint check chi tiết đến từng đối tượng, cần cấu hình các endpoin theo chuẩn như:  

| URL                                      | Ý nghĩa / Mục đích                                                                           |
| :--------------------------------------- | :------------------------------------------------------------------------------------------- |
| `/health`                                | Tổng quan: service còn sống hay không (Healthy/Unhealthy).                                   |
| `/health/ready` hoặc `/health/readiness` | **Readiness**: Service đã **sẵn sàng** nhận request chưa. (Ví dụ: database đã connect chưa.) |
| `/health/live` hoặc `/health/liveness`   | **Liveness**: Service còn **sống** không (ví dụ không bị crash, deadlock).                   |



<pre style="font-family:Cascadia Mono;font-size:13px;color:#dadada;background:#1e1e1e;"><span style="color:#d7ba7d;">&quot;HealthCheckUrl&quot;</span><span style="color:#b4b4b4;">:</span>&nbsp;<span style="color:#b4b4b4;">{</span>
&nbsp;&nbsp;<span style="color:#d7ba7d;">&quot;KafkaFOX&quot;</span><span style="color:#b4b4b4;">:</span>&nbsp;<span style="color:#d69d85;">&quot;/health/kafka-fox&quot;</span><span style="color:#b4b4b4;">,</span>
&nbsp;&nbsp;<span style="color:#d7ba7d;">&quot;AccountDb&quot;</span><span style="color:#b4b4b4;">:</span>&nbsp;<span style="color:#d69d85;">&quot;/health/account-db&quot;</span><span style="color:#b4b4b4;">,</span>
&nbsp;&nbsp;<span style="color:#d7ba7d;">&quot;NovaDb&quot;</span><span style="color:#b4b4b4;">:</span>&nbsp;<span style="color:#d69d85;">&quot;/health/nova-db&quot;</span><span style="color:#b4b4b4;">,</span>
&nbsp;&nbsp;<span style="color:#d7ba7d;">&quot;Health&quot;</span><span style="color:#b4b4b4;">:</span>&nbsp;<span style="color:#d69d85;">&quot;/health&quot;</span><span style="color:#b4b4b4;">,</span>&nbsp;<span style="color:#57a64a;">//=&gt;&nbsp;Check&nbsp;full&nbsp;thông&nbsp;tin&nbsp;healcheck</span>
&nbsp;&nbsp;<span style="color:#d7ba7d;">&quot;Live&quot;</span><span style="color:#b4b4b4;">:</span>&nbsp;<span style="color:#d69d85;">&quot;/health/live&quot;</span><span style="color:#b4b4b4;">,</span>&nbsp;<span style="color:#57a64a;">//=&gt;&nbsp;Dùng&nbsp;để&nbsp;các&nbsp;service&nbsp;khác&nbsp;cần&nbsp;healcheck&nbsp;đến&nbsp;mình</span>
&nbsp;&nbsp;<span style="color:#d7ba7d;">&quot;Ready&quot;</span><span style="color:#b4b4b4;">:</span>&nbsp;<span style="color:#d69d85;">&quot;/health/ready&quot;</span><span style="color:#b4b4b4;">,</span>&nbsp;<span style="color:#57a64a;">//=&gt;&nbsp;Check&nbsp;các&nbsp;connect&nbsp;cần&nbsp;thiết&nbsp;để&nbsp;service&nbsp;hoạt&nbsp;động&nbsp;đúng</span>
&nbsp;&nbsp;<span style="color:#d7ba7d;">&quot;Metric&quot;</span><span style="color:#b4b4b4;">:</span>&nbsp;<span style="color:#d69d85;">&quot;/metrics&quot;</span>&nbsp;<span style="color:#57a64a;">//=&gt;&nbsp;Dùng&nbsp;để&nbsp;cung&nbsp;cấp&nbsp;cho&nbsp;team&nbsp;hạ&nbsp;tầng&nbsp;cấu&nbsp;hình&nbsp;healthcheck</span>
<span style="color:#b4b4b4;">}</span></pre>

- Tạo class để map phần setting này:   
(chỉ cần map các url nào cần)

<pre style="font-family:Cascadia Mono;font-size:13px;color:#dadada;background:#1e1e1e;"><span style="color:#569cd6;">public</span>&nbsp;<span style="color:#569cd6;">class</span>&nbsp;<span style="color:#4ec9b0;">HealthCheckUrl</span>
<span style="color:gainsboro;">{</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">public</span>&nbsp;<span style="color:#569cd6;">string</span>&nbsp;<span style="color:gainsboro;">KafkaFOX</span>&nbsp;<span style="color:gainsboro;">{</span>&nbsp;<span style="color:#569cd6;">get</span><span style="color:gainsboro;">;</span>&nbsp;<span style="color:#569cd6;">set</span><span style="color:gainsboro;">;</span>&nbsp;<span style="color:gainsboro;">}</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#608b4e;">///</span><span style="color:#608b4e;">&nbsp;</span><span style="color:#608b4e;">&lt;</span><span style="color:#608b4e;">summary</span><span style="color:#608b4e;">&gt;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#608b4e;">///</span><span style="color:#608b4e;">&nbsp;Check&nbsp;full&nbsp;thông&nbsp;tin</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#608b4e;">///</span><span style="color:#608b4e;">&nbsp;</span><span style="color:#608b4e;">&lt;/</span><span style="color:#608b4e;">summary</span><span style="color:#608b4e;">&gt;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">public</span>&nbsp;<span style="color:#569cd6;">string</span>&nbsp;<span style="color:gainsboro;">Health</span>&nbsp;<span style="color:gainsboro;">{</span>&nbsp;<span style="color:#569cd6;">get</span><span style="color:gainsboro;">;</span>&nbsp;<span style="color:#569cd6;">set</span><span style="color:gainsboro;">;</span>&nbsp;<span style="color:gainsboro;">}</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#608b4e;">///</span><span style="color:#608b4e;">&nbsp;</span><span style="color:#608b4e;">&lt;</span><span style="color:#608b4e;">summary</span><span style="color:#608b4e;">&gt;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#608b4e;">///</span><span style="color:#608b4e;">&nbsp;Chỉ&nbsp;check&nbsp;sv&nbsp;đang&nbsp;sống</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#608b4e;">///</span><span style="color:#608b4e;">&nbsp;</span><span style="color:#608b4e;">&lt;/</span><span style="color:#608b4e;">summary</span><span style="color:#608b4e;">&gt;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">public</span>&nbsp;<span style="color:#569cd6;">string</span>&nbsp;<span style="color:gainsboro;">Live</span>&nbsp;<span style="color:gainsboro;">{</span>&nbsp;<span style="color:#569cd6;">get</span><span style="color:gainsboro;">;</span>&nbsp;<span style="color:#569cd6;">set</span><span style="color:gainsboro;">;</span>&nbsp;<span style="color:gainsboro;">}</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#608b4e;">///</span><span style="color:#608b4e;">&nbsp;</span><span style="color:#608b4e;">&lt;</span><span style="color:#608b4e;">summary</span><span style="color:#608b4e;">&gt;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#608b4e;">///</span><span style="color:#608b4e;">&nbsp;Check&nbsp;sv&nbsp;sẵn&nbsp;sàng&nbsp;xử&nbsp;lý</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#608b4e;">///</span><span style="color:#608b4e;">&nbsp;</span><span style="color:#608b4e;">&lt;/</span><span style="color:#608b4e;">summary</span><span style="color:#608b4e;">&gt;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">public</span>&nbsp;<span style="color:#569cd6;">string</span>&nbsp;<span style="color:gainsboro;">Ready</span>&nbsp;<span style="color:gainsboro;">{</span>&nbsp;<span style="color:#569cd6;">get</span><span style="color:gainsboro;">;</span>&nbsp;<span style="color:#569cd6;">set</span><span style="color:gainsboro;">;</span>&nbsp;<span style="color:gainsboro;">}</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">public</span>&nbsp;<span style="color:#569cd6;">string</span>&nbsp;<span style="color:gainsboro;">AccountDb</span>&nbsp;<span style="color:gainsboro;">{</span>&nbsp;<span style="color:#569cd6;">get</span><span style="color:gainsboro;">;</span>&nbsp;<span style="color:#569cd6;">set</span><span style="color:gainsboro;">;</span>&nbsp;<span style="color:gainsboro;">}</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">public</span>&nbsp;<span style="color:#569cd6;">string</span>&nbsp;<span style="color:gainsboro;">TradingDb</span>&nbsp;<span style="color:gainsboro;">{</span>&nbsp;<span style="color:#569cd6;">get</span><span style="color:gainsboro;">;</span>&nbsp;<span style="color:#569cd6;">set</span><span style="color:gainsboro;">;</span>&nbsp;<span style="color:gainsboro;">}</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">public</span>&nbsp;<span style="color:#569cd6;">string</span>&nbsp;<span style="color:gainsboro;">NovaDb</span>&nbsp;<span style="color:gainsboro;">{</span>&nbsp;<span style="color:#569cd6;">get</span><span style="color:gainsboro;">;</span>&nbsp;<span style="color:#569cd6;">set</span><span style="color:gainsboro;">;</span>&nbsp;<span style="color:gainsboro;">}</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">public</span>&nbsp;<span style="color:#569cd6;">string</span>&nbsp;<span style="color:gainsboro;">MainSv</span>&nbsp;<span style="color:gainsboro;">{</span>&nbsp;<span style="color:#569cd6;">get</span><span style="color:gainsboro;">;</span>&nbsp;<span style="color:#569cd6;">set</span><span style="color:gainsboro;">;</span>&nbsp;<span style="color:gainsboro;">}</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">public</span>&nbsp;<span style="color:#569cd6;">string</span>&nbsp;<span style="color:gainsboro;">Metric</span>&nbsp;<span style="color:gainsboro;">{</span>&nbsp;<span style="color:#569cd6;">get</span><span style="color:gainsboro;">;</span>&nbsp;<span style="color:#569cd6;">set</span><span style="color:gainsboro;">;</span>&nbsp;<span style="color:gainsboro;">}</span>
 
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">public</span>&nbsp;<span style="color:#569cd6;">string</span>&nbsp;<span style="color:#dcdcaa;">CreateMainSvUrl</span><span style="color:gainsboro;">(</span><span style="color:#569cd6;">int</span>&nbsp;<span style="color:#9cdcfe;">instanceNumber</span><span style="color:gainsboro;">)</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#b4b4b4;">=&gt;</span>&nbsp;<span style="color:#d69d85;">$&quot;</span><span style="color:gainsboro;">{</span><span style="color:gainsboro;">MainSv</span><span style="color:gainsboro;">}</span><span style="color:#d69d85;">-</span><span style="color:gainsboro;">{</span><span style="color:#9cdcfe;">instanceNumber</span><span style="color:gainsboro;">}</span><span style="color:#d69d85;">&quot;</span><span style="color:gainsboro;">;</span>
<span style="color:gainsboro;">}</span></pre>


- Class chưa các tag cần thiết, dùng để đánh dấu cho từng connect cần healthcheck:

<pre style="font-family:Cascadia Mono;font-size:13px;color:#dadada;background:#1e1e1e;"><span style="color:#569cd6;">public</span>&nbsp;<span style="color:#569cd6;">static</span>&nbsp;<span style="color:#569cd6;">class</span>&nbsp;<span style="color:#4ec9b0;">HealthCheckTagName</span>
<span style="color:gainsboro;">{</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">public</span>&nbsp;<span style="color:#569cd6;">const</span>&nbsp;<span style="color:#569cd6;">string</span>&nbsp;<span style="color:gainsboro;">Kafka_FOX</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#d69d85;">&quot;kafka-fox&quot;</span><span style="color:gainsboro;">;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">public</span>&nbsp;<span style="color:#569cd6;">const</span>&nbsp;<span style="color:#569cd6;">string</span>&nbsp;<span style="color:gainsboro;">AccountDb</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#d69d85;">&quot;account-db&quot;</span><span style="color:gainsboro;">;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">public</span>&nbsp;<span style="color:#569cd6;">const</span>&nbsp;<span style="color:#569cd6;">string</span>&nbsp;<span style="color:gainsboro;">TradingDb</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#d69d85;">&quot;trading-db&quot;</span><span style="color:gainsboro;">;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">public</span>&nbsp;<span style="color:#569cd6;">const</span>&nbsp;<span style="color:#569cd6;">string</span>&nbsp;<span style="color:gainsboro;">NovaDb</span>&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#d69d85;">&quot;nova-db&quot;</span><span style="color:gainsboro;">;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#608b4e;">///</span><span style="color:#608b4e;">&nbsp;</span><span style="color:#608b4e;">&lt;</span><span style="color:#608b4e;">summary</span><span style="color:#608b4e;">&gt;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#608b4e;">///</span><span style="color:#608b4e;">&nbsp;Check&nbsp;service&nbsp;sống&nbsp;(cho&nbsp;các&nbsp;service&nbsp;khác&nbsp;check)</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#608b4e;">///</span><span style="color:#608b4e;">&nbsp;</span><span style="color:#608b4e;">&lt;/</span><span style="color:#608b4e;">summary</span><span style="color:#608b4e;">&gt;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">public</span>&nbsp;<span style="color:#569cd6;">const</span>&nbsp;<span style="color:#569cd6;">string</span>&nbsp;<span style="color:gainsboro;">Live</span>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#d69d85;">&quot;live&quot;</span><span style="color:gainsboro;">;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#608b4e;">///</span><span style="color:#608b4e;">&nbsp;</span><span style="color:#608b4e;">&lt;</span><span style="color:#608b4e;">summary</span><span style="color:#608b4e;">&gt;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#608b4e;">///</span><span style="color:#608b4e;">&nbsp;Check&nbsp;service&nbsp;sẵn&nbsp;sàng</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#608b4e;">///</span><span style="color:#608b4e;">&nbsp;</span><span style="color:#608b4e;">&lt;/</span><span style="color:#608b4e;">summary</span><span style="color:#608b4e;">&gt;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">public</span>&nbsp;<span style="color:#569cd6;">const</span>&nbsp;<span style="color:#569cd6;">string</span>&nbsp;<span style="color:gainsboro;">Ready</span>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#d69d85;">&quot;ready&quot;</span><span style="color:gainsboro;">;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#608b4e;">///</span><span style="color:#608b4e;">&nbsp;</span><span style="color:#608b4e;">&lt;</span><span style="color:#608b4e;">summary</span><span style="color:#608b4e;">&gt;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#608b4e;">///</span><span style="color:#608b4e;">&nbsp;Check&nbsp;thông&nbsp;tin&nbsp;đến&nbsp;main&nbsp;service&nbsp;(base)</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#608b4e;">///</span><span style="color:#608b4e;">&nbsp;</span><span style="color:#608b4e;">&lt;/</span><span style="color:#608b4e;">summary</span><span style="color:#608b4e;">&gt;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">public</span>&nbsp;<span style="color:#569cd6;">const</span>&nbsp;<span style="color:#569cd6;">string</span>&nbsp;<span style="color:gainsboro;">MainSv</span>&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#d69d85;">&quot;main-sv&quot;</span><span style="color:gainsboro;">;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#608b4e;">///</span><span style="color:#608b4e;">&nbsp;</span><span style="color:#608b4e;">&lt;</span><span style="color:#608b4e;">summary</span><span style="color:#608b4e;">&gt;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#608b4e;">///</span><span style="color:#608b4e;">&nbsp;Hàm&nbsp;tạo&nbsp;tag&nbsp;cho&nbsp;từng&nbsp;main&nbsp;service</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#608b4e;">///</span><span style="color:#608b4e;">&nbsp;</span><span style="color:#608b4e;">&lt;/</span><span style="color:#608b4e;">summary</span><span style="color:#608b4e;">&gt;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#608b4e;">///</span><span style="color:#608b4e;">&nbsp;</span><span style="color:#608b4e;">&lt;</span><span style="color:#608b4e;">param</span>&nbsp;<span style="color:#c8c8c8;">name</span><span style="color:#608b4e;">=</span><span style="color:#c8c8c8;">&quot;</span><span style="color:#9cdcfe;">instanceNumber</span><span style="color:#c8c8c8;">&quot;</span><span style="color:#608b4e;">&gt;</span><span style="color:#608b4e;">&lt;/</span><span style="color:#608b4e;">param</span><span style="color:#608b4e;">&gt;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#608b4e;">///</span><span style="color:#608b4e;">&nbsp;</span><span style="color:#608b4e;">&lt;</span><span style="color:#608b4e;">returns</span><span style="color:#608b4e;">&gt;</span><span style="color:#608b4e;">&lt;/</span><span style="color:#608b4e;">returns</span><span style="color:#608b4e;">&gt;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">public</span>&nbsp;<span style="color:#569cd6;">static</span>&nbsp;<span style="color:#569cd6;">string</span>&nbsp;<span style="color:#dcdcaa;">CreateMainSvTag</span><span style="color:gainsboro;">(</span><span style="color:#569cd6;">int</span>&nbsp;<span style="color:#9cdcfe;">instanceNumber</span><span style="color:gainsboro;">)</span>&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#b4b4b4;">=&gt;</span>&nbsp;<span style="color:#d69d85;">$&quot;</span><span style="color:gainsboro;">{</span><span style="color:gainsboro;">MainSv</span><span style="color:gainsboro;">}</span><span style="color:#d69d85;">-</span><span style="color:gainsboro;">{</span><span style="color:#9cdcfe;">instanceNumber</span><span style="color:gainsboro;">}</span><span style="color:#d69d85;">&quot;</span><span style="color:gainsboro;">;</span>
<span style="color:gainsboro;">}</span></pre>

---

### Add Healcheck
- Add các healthcheck, lưu ý một số thuật ngữ:
    - Tag: 
        - Mỗi connect có thể gắn 1 hoặc nhiều tag => Dùng để gom nhóm vào kéo vào các url healthcheck riêng
        - VD: có thể gom các healcheck quan trọng để xác định service ready, và gom vào 1 url chung
    - Name: là name các healthcheck sẽ hiển thị trên /metrics


<pre style="font-family:Cascadia Mono;font-size:13px;color:#dadada;background:#1e1e1e;"><span style="color:#569cd6;">public</span>&nbsp;<span style="color:#569cd6;">static</span>&nbsp;<span style="color:#b8d7a3;">IServiceCollection</span>&nbsp;<span style="color:#dcdcaa;">AddHealthChecks</span><span style="color:gainsboro;">(</span><span style="color:#569cd6;">this</span>&nbsp;<span style="color:#b8d7a3;">IServiceCollection</span>&nbsp;<span style="color:#9cdcfe;">services</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#b8d7a3;">IConfiguration</span>&nbsp;<span style="color:#9cdcfe;">configuration</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#4ec9b0;">AppSetting</span>&nbsp;<span style="color:#9cdcfe;">appSetting</span><span style="color:gainsboro;">)</span>
<span style="color:gainsboro;">{</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#57a64a;">//Lấy&nbsp;các&nbsp;nguyên&nbsp;liệu&nbsp;môi&nbsp;trường&nbsp;của&nbsp;các&nbsp;đối&nbsp;tượng&nbsp;cần&nbsp;health&nbsp;check</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">var</span>&nbsp;<span style="color:#9cdcfe;">bootstrapServerFox</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#9cdcfe;">appSetting</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">LoggingSetting</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">BootstrapServers</span><span style="color:gainsboro;">;</span>&nbsp;<span style="color:#57a64a;">//FOX</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">var</span>&nbsp;<span style="color:#9cdcfe;">accountAppSetting</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#9cdcfe;">appSetting</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">AccountAppSetting</span><span style="color:gainsboro;">;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">var</span>&nbsp;<span style="color:#9cdcfe;">tradingDbConnectString</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#9cdcfe;">configuration</span><span style="color:gainsboro;">[</span><span style="color:#d69d85;">&quot;DBConnectionString:TRADING&quot;</span><span style="color:gainsboro;">]</span><span style="color:gainsboro;">;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">var</span>&nbsp;<span style="color:#9cdcfe;">novaDbConnectString</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#9cdcfe;">configuration</span><span style="color:gainsboro;">[</span><span style="color:#d69d85;">&quot;DBConnectionString:NOVA&quot;</span><span style="color:gainsboro;">]</span><span style="color:gainsboro;">;</span>
 
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">var</span>&nbsp;<span style="color:#9cdcfe;">producerConfigKafkaFOX</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#569cd6;">new</span>&nbsp;<span style="color:#4ec9b0;">ProducerConfig</span><span style="color:gainsboro;">(</span><span style="color:gainsboro;">)</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">{</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">BootstrapServers</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#9cdcfe;">bootstrapServerFox</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">}</span><span style="color:gainsboro;">;</span>
 
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#57a64a;">//Đăng&nbsp;ký&nbsp;health&nbsp;check&nbsp;cho&nbsp;các&nbsp;dịch&nbsp;vụ</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#9cdcfe;">services</span><span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">AddHealthChecks</span><span style="color:gainsboro;">(</span><span style="color:gainsboro;">)</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">AddCheck</span><span style="color:gainsboro;">(</span><span style="color:#4ec9b0;">HealthCheckTagName</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">Live</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:gainsboro;">(</span><span style="color:gainsboro;">)</span>&nbsp;<span style="color:#b4b4b4;">=&gt;</span>&nbsp;<span style="color:#86c691;">HealthCheckResult</span><span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">Healthy</span><span style="color:gainsboro;">(</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#9cdcfe;">tags</span><span style="color:gainsboro;">:</span>&nbsp;<span style="color:#569cd6;">new</span><span style="color:gainsboro;">[</span><span style="color:gainsboro;">]</span>&nbsp;<span style="color:gainsboro;">{</span>&nbsp;<span style="color:#4ec9b0;">HealthCheckTagName</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">Live</span><span style="color:gainsboro;">}</span><span style="color:gainsboro;">)</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">AddKafka</span><span style="color:gainsboro;">(</span><span style="color:#9cdcfe;">producerConfigKafkaFOX</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#9cdcfe;">tags</span><span style="color:gainsboro;">:</span>&nbsp;<span style="color:#569cd6;">new</span>&nbsp;<span style="color:#569cd6;">string</span><span style="color:gainsboro;">[</span><span style="color:gainsboro;">]</span>&nbsp;<span style="color:gainsboro;">{</span>&nbsp;<span style="color:#4ec9b0;">HealthCheckTagName</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">Kafka_FOX</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#4ec9b0;">HealthCheckTagName</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">Ready</span>&nbsp;<span style="color:gainsboro;">}</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#9cdcfe;">name</span><span style="color:gainsboro;">:</span>&nbsp;<span style="color:#4ec9b0;">HealthCheckTagName</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">Kafka_FOX</span><span style="color:gainsboro;">)</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">AddOracle</span><span style="color:gainsboro;">(</span><span style="color:#9cdcfe;">appSetting</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">DBConnectionString</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#9cdcfe;">tags</span><span style="color:gainsboro;">:</span>&nbsp;<span style="color:#569cd6;">new</span>&nbsp;<span style="color:#569cd6;">string</span><span style="color:gainsboro;">[</span><span style="color:gainsboro;">]</span>&nbsp;<span style="color:gainsboro;">{</span>&nbsp;<span style="color:#4ec9b0;">HealthCheckTagName</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">AccountDb</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#4ec9b0;">HealthCheckTagName</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">Ready</span>&nbsp;<span style="color:gainsboro;">}</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#9cdcfe;">name</span><span style="color:gainsboro;">:</span>&nbsp;<span style="color:#4ec9b0;">HealthCheckTagName</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">AccountDb</span><span style="color:gainsboro;">)</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">AddOracle</span><span style="color:gainsboro;">(</span><span style="color:#9cdcfe;">tradingDbConnectString</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#9cdcfe;">tags</span><span style="color:gainsboro;">:</span>&nbsp;<span style="color:#569cd6;">new</span>&nbsp;<span style="color:#569cd6;">string</span><span style="color:gainsboro;">[</span><span style="color:gainsboro;">]</span>&nbsp;<span style="color:gainsboro;">{</span>&nbsp;<span style="color:#4ec9b0;">HealthCheckTagName</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">TradingDb</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#4ec9b0;">HealthCheckTagName</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">Ready</span>&nbsp;<span style="color:gainsboro;">}</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#9cdcfe;">name</span><span style="color:gainsboro;">:</span>&nbsp;<span style="color:#4ec9b0;">HealthCheckTagName</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">TradingDb</span><span style="color:gainsboro;">)</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">AddOracle</span><span style="color:gainsboro;">(</span><span style="color:#9cdcfe;">novaDbConnectString</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#9cdcfe;">tags</span><span style="color:gainsboro;">:</span>&nbsp;<span style="color:#569cd6;">new</span>&nbsp;<span style="color:#569cd6;">string</span><span style="color:gainsboro;">[</span><span style="color:gainsboro;">]</span>&nbsp;<span style="color:gainsboro;">{</span>&nbsp;<span style="color:#4ec9b0;">HealthCheckTagName</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">NovaDb</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#4ec9b0;">HealthCheckTagName</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">Ready</span>&nbsp;<span style="color:gainsboro;">}</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#9cdcfe;">name</span><span style="color:gainsboro;">:</span>&nbsp;<span style="color:#4ec9b0;">HealthCheckTagName</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">NovaDb</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">;</span>
&nbsp;&nbsp;&nbsp;&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#57a64a;">//add&nbsp;healcheck&nbsp;đến&nbsp;main&nbsp;service</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#57a64a;">//Chỉ&nbsp;dùng&nbsp;trong&nbsp;trường&nbsp;hợp&nbsp;service&nbsp;A&nbsp;cần&nbsp;check&nbsp;connect&nbsp;đến&nbsp;service&nbsp;B&nbsp;và&nbsp;service&nbsp;B&nbsp;có&nbsp;nhiều&nbsp;hơn&nbsp;1&nbsp;instance,&nbsp;và&nbsp;các&nbsp;instance&nbsp;độc&nbsp;lập&nbsp;nhau</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#d8a0df;">for</span>&nbsp;<span style="color:gainsboro;">(</span><span style="color:#569cd6;">int</span>&nbsp;<span style="color:#9cdcfe;">i</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#b5cea8;">1</span><span style="color:gainsboro;">;</span>&nbsp;<span style="color:#9cdcfe;">i</span>&nbsp;<span style="color:#b4b4b4;">&lt;=</span>&nbsp;<span style="color:#9cdcfe;">accountAppSetting</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">TotalInstances</span><span style="color:gainsboro;">;</span>&nbsp;<span style="color:#9cdcfe;">i</span><span style="color:#b4b4b4;">++</span><span style="color:gainsboro;">)</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">{</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">string</span>&nbsp;<span style="color:#9cdcfe;">uri</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#569cd6;">string</span><span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">Format</span><span style="color:gainsboro;">(</span><span style="color:#9cdcfe;">accountAppSetting</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">BaseUrl</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#9cdcfe;">i</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">;</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">string</span>&nbsp;<span style="color:#9cdcfe;">tagMainBase</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#4ec9b0;">HealthCheckTagName</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">MainSv</span><span style="color:gainsboro;">;</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">string</span>&nbsp;<span style="color:#9cdcfe;">tagMainByInstance</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#4ec9b0;">HealthCheckTagName</span><span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">CreateMainSvTag</span><span style="color:gainsboro;">(</span><span style="color:#9cdcfe;">i</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">;</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#9cdcfe;">services</span><span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">AddHealthChecks</span><span style="color:gainsboro;">(</span><span style="color:gainsboro;">)</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">AddUrlGroup</span><span style="color:gainsboro;">(</span><span style="color:#569cd6;">new</span>&nbsp;<span style="color:#4ec9b0;">Uri</span><span style="color:gainsboro;">(</span><span style="color:#d69d85;">$&quot;</span><span style="color:gainsboro;">{</span><span style="color:#9cdcfe;">uri</span><span style="color:gainsboro;">}</span><span style="color:#d69d85;">/health/live</span><span style="color:#d69d85;">&quot;</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#9cdcfe;">tags</span><span style="color:gainsboro;">:</span>&nbsp;<span style="color:#569cd6;">new</span>&nbsp;<span style="color:#569cd6;">string</span><span style="color:gainsboro;">[</span><span style="color:gainsboro;">]</span>&nbsp;<span style="color:gainsboro;">{</span>&nbsp;<span style="color:#9cdcfe;">tagMainByInstance</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#9cdcfe;">tagMainBase</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#4ec9b0;">HealthCheckTagName</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">Ready</span>&nbsp;<span style="color:gainsboro;">}</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#9cdcfe;">name</span><span style="color:gainsboro;">:</span>&nbsp;<span style="color:#9cdcfe;">tagMainByInstance</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">}</span>
 
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#d8a0df;">return</span>&nbsp;<span style="color:#9cdcfe;">services</span><span style="color:gainsboro;">;</span>
<span style="color:gainsboro;">}</span></pre>

- Nhớ gọi hàm đăng ký:
<pre style="font-family:Cascadia Mono;font-size:13px;color:#dadada;background:#1e1e1e;"><span style="color:#9cdcfe;">builder</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">Services</span><span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">AddHealthChecks</span><span style="color:gainsboro;">(</span><span style="color:#9cdcfe;">builder</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">Configuration</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#9cdcfe;">appSetting</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">;</span>
</pre>

---

## Map healthcheck tạo các endpoint
- Mục đích: 
    - Map các url ứng với các tag healthcheck nào
    - Expose ra prometheus

- Tạo class riêng map các endpoint healthcheck cho chuẩn
<pre style="font-family:Cascadia Mono;font-size:13px;color:#dadada;background:#1e1e1e;"><span style="color:#608b4e;">///</span><span style="color:#608b4e;">&nbsp;</span><span style="color:#608b4e;">&lt;</span><span style="color:#608b4e;">summary</span><span style="color:#608b4e;">&gt;</span>
<span style="color:#608b4e;">///</span><span style="color:#608b4e;">&nbsp;Cấu&nbsp;hình&nbsp;các&nbsp;endpoin&nbsp;cho&nbsp;healcheck</span>
<span style="color:#608b4e;">///</span><span style="color:#608b4e;">&nbsp;</span><span style="color:#608b4e;">&lt;/</span><span style="color:#608b4e;">summary</span><span style="color:#608b4e;">&gt;</span>
<span style="color:#608b4e;">///</span><span style="color:#608b4e;">&nbsp;</span><span style="color:#608b4e;">&lt;</span><span style="color:#608b4e;">param</span>&nbsp;<span style="color:#c8c8c8;">name</span><span style="color:#608b4e;">=</span><span style="color:#c8c8c8;">&quot;</span><span style="color:#9cdcfe;">app</span><span style="color:#c8c8c8;">&quot;</span><span style="color:#608b4e;">&gt;</span><span style="color:#608b4e;">&lt;/</span><span style="color:#608b4e;">param</span><span style="color:#608b4e;">&gt;</span>
<span style="color:#569cd6;">public</span>&nbsp;<span style="color:#569cd6;">static</span>&nbsp;<span style="color:#569cd6;">void</span>&nbsp;<span style="color:#dcdcaa;">ConfigureHealthChecks</span><span style="color:gainsboro;">(</span><span style="color:#569cd6;">this</span>&nbsp;<span style="color:#4ec9b0;">WebApplication</span>&nbsp;<span style="color:#9cdcfe;">app</span><span style="color:gainsboro;">)</span>
<span style="color:gainsboro;">{</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">var</span>&nbsp;<span style="color:#9cdcfe;">config</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#9cdcfe;">app</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">Configuration</span><span style="color:gainsboro;">;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">var</span>&nbsp;<span style="color:#9cdcfe;">healthCheckUrl</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#9cdcfe;">config</span><span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">GetSection</span><span style="color:gainsboro;">(</span><span style="color:#569cd6;">nameof</span><span style="color:gainsboro;">(</span><span style="color:#4ec9b0;">HealthCheckUrl</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">)</span><span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">Get</span><span style="color:gainsboro;">&lt;</span><span style="color:#4ec9b0;">HealthCheckUrl</span><span style="color:gainsboro;">&gt;</span><span style="color:gainsboro;">(</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">var</span>&nbsp;<span style="color:#9cdcfe;">appSetting</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#9cdcfe;">app</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">Services</span><span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">GetRequiredService</span><span style="color:gainsboro;">&lt;</span><span style="color:#4ec9b0;">AppSetting</span><span style="color:gainsboro;">&gt;</span><span style="color:gainsboro;">(</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">var</span>&nbsp;<span style="color:#9cdcfe;">accountAppSetting</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#9cdcfe;">appSetting</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">AccountAppSetting</span><span style="color:gainsboro;">;</span>
 
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">var</span>&nbsp;<span style="color:#9cdcfe;">checks</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#569cd6;">new</span>&nbsp;<span style="color:gainsboro;">(</span><span style="color:#569cd6;">string</span>&nbsp;<span style="color:gainsboro;">Url</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#569cd6;">string</span>&nbsp;<span style="color:gainsboro;">Tag</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">[</span><span style="color:gainsboro;">]</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">{</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">(</span><span style="color:#9cdcfe;">healthCheckUrl</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">KafkaFOX</span><span style="color:gainsboro;">,</span>&nbsp;&nbsp;<span style="color:#4ec9b0;">HealthCheckTagName</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">Kafka_FOX</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">,</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">(</span><span style="color:#9cdcfe;">healthCheckUrl</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">AccountDb</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#4ec9b0;">HealthCheckTagName</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">AccountDb</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">,</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">(</span><span style="color:#9cdcfe;">healthCheckUrl</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">TradingDb</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#4ec9b0;">HealthCheckTagName</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">TradingDb</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">,</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">(</span><span style="color:#9cdcfe;">healthCheckUrl</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">NovaDb</span><span style="color:gainsboro;">,</span>&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#4ec9b0;">HealthCheckTagName</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">NovaDb</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">,</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">(</span><span style="color:#9cdcfe;">healthCheckUrl</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">MainSv</span><span style="color:gainsboro;">,</span>&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#4ec9b0;">HealthCheckTagName</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">MainSv</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">,</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">(</span><span style="color:#9cdcfe;">healthCheckUrl</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">Ready</span><span style="color:gainsboro;">,</span>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#4ec9b0;">HealthCheckTagName</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">Ready</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">,</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">(</span><span style="color:#9cdcfe;">healthCheckUrl</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">Live</span><span style="color:gainsboro;">,</span>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#4ec9b0;">HealthCheckTagName</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">Live</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">,</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">}</span><span style="color:gainsboro;">;</span>
 
 
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#57a64a;">//&nbsp;Group&nbsp;by&nbsp;URL&nbsp;(trong&nbsp;trường&nbsp;hợp&nbsp;1&nbsp;url&nbsp;muốn&nbsp;nhiều&nbsp;tag)</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">var</span>&nbsp;<span style="color:#9cdcfe;">urlTagMap</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#9cdcfe;">checks</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">GroupBy</span><span style="color:gainsboro;">(</span><span style="color:#9cdcfe;">c</span>&nbsp;<span style="color:#b4b4b4;">=&gt;</span>&nbsp;<span style="color:#9cdcfe;">c</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">Url</span><span style="color:gainsboro;">)</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">ToDictionary</span><span style="color:gainsboro;">(</span><span style="color:#9cdcfe;">g</span>&nbsp;<span style="color:#b4b4b4;">=&gt;</span>&nbsp;<span style="color:#9cdcfe;">g</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">Key</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#9cdcfe;">g</span>&nbsp;<span style="color:#b4b4b4;">=&gt;</span>&nbsp;<span style="color:#9cdcfe;">g</span><span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">Select</span><span style="color:gainsboro;">(</span><span style="color:#9cdcfe;">x</span>&nbsp;<span style="color:#b4b4b4;">=&gt;</span>&nbsp;<span style="color:#9cdcfe;">x</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">Tag</span><span style="color:gainsboro;">)</span><span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">ToList</span><span style="color:gainsboro;">(</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">;</span>
 
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#57a64a;">//Add&nbsp;các&nbsp;endpoin&nbsp;healcheck&nbsp;tổng&nbsp;hợp</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#57a64a;">//Với&nbsp;main&nbsp;sẽ&nbsp;add&nbsp;1&nbsp;endpoin&nbsp;tổng&nbsp;hợp</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#d8a0df;">foreach</span>&nbsp;<span style="color:gainsboro;">(</span><span style="color:#569cd6;">var</span>&nbsp;<span style="color:gainsboro;">(</span><span style="color:#9cdcfe;">url</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#9cdcfe;">tags</span><span style="color:gainsboro;">)</span>&nbsp;<span style="color:#d8a0df;">in</span>&nbsp;<span style="color:#9cdcfe;">urlTagMap</span><span style="color:gainsboro;">)</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">{</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#9cdcfe;">app</span><span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">UseHealthChecks</span><span style="color:gainsboro;">(</span><span style="color:#9cdcfe;">url</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#569cd6;">new</span>&nbsp;<span style="color:#4ec9b0;">HealthCheckOptions</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">{</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">Predicate</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#9cdcfe;">r</span>&nbsp;<span style="color:#b4b4b4;">=&gt;</span>&nbsp;<span style="color:#9cdcfe;">r</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">Tags</span><span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">Any</span><span style="color:gainsboro;">(</span><span style="color:#9cdcfe;">tag</span>&nbsp;<span style="color:#b4b4b4;">=&gt;</span>&nbsp;<span style="color:#9cdcfe;">tags</span><span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">Contains</span><span style="color:gainsboro;">(</span><span style="color:#9cdcfe;">tag</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">,</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">ResponseWriter</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#4ec9b0;">UIResponseWriter</span><span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">WriteHealthCheckUIResponse</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">}</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">}</span>
 
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#57a64a;">//Add&nbsp;riêng&nbsp;cho&nbsp;phần&nbsp;healcheck&nbsp;đến&nbsp;chi&nbsp;tiết&nbsp;từng&nbsp;main&nbsp;service</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#d8a0df;">for</span>&nbsp;<span style="color:gainsboro;">(</span><span style="color:#569cd6;">int</span>&nbsp;<span style="color:#9cdcfe;">i</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#b5cea8;">1</span><span style="color:gainsboro;">;</span>&nbsp;<span style="color:#9cdcfe;">i</span>&nbsp;<span style="color:#b4b4b4;">&lt;=</span>&nbsp;<span style="color:#9cdcfe;">accountAppSetting</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">TotalInstances</span><span style="color:gainsboro;">;</span>&nbsp;<span style="color:#9cdcfe;">i</span><span style="color:#b4b4b4;">++</span><span style="color:gainsboro;">)</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">{</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">string</span>&nbsp;<span style="color:#9cdcfe;">urlInstance</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#9cdcfe;">healthCheckUrl</span><span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">CreateMainSvUrl</span><span style="color:gainsboro;">(</span><span style="color:#9cdcfe;">i</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">;</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#569cd6;">string</span>&nbsp;<span style="color:#9cdcfe;">tagInstance</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#4ec9b0;">HealthCheckTagName</span><span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">CreateMainSvTag</span><span style="color:gainsboro;">(</span><span style="color:#9cdcfe;">i</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">;</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#9cdcfe;">app</span><span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">UseHealthChecks</span><span style="color:gainsboro;">(</span><span style="color:#9cdcfe;">urlInstance</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#569cd6;">new</span>&nbsp;<span style="color:#4ec9b0;">HealthCheckOptions</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">{</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">Predicate</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#9cdcfe;">r</span>&nbsp;<span style="color:#b4b4b4;">=&gt;</span>&nbsp;<span style="color:#9cdcfe;">r</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">Tags</span><span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">Contains</span><span style="color:gainsboro;">(</span><span style="color:#9cdcfe;">tagInstance</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">,</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">ResponseWriter</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#4ec9b0;">UIResponseWriter</span><span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">WriteHealthCheckUIResponse</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">}</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">}</span>
 
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#57a64a;">//xuất&nbsp;trạng&nbsp;thái&nbsp;health&nbsp;check&nbsp;của&nbsp;các&nbsp;service&nbsp;dưới&nbsp;dạng&nbsp;metric&nbsp;Prometheus</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#9cdcfe;">app</span><span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">UseHealthChecksPrometheusExporter</span><span style="color:gainsboro;">(</span><span style="color:#9cdcfe;">healthCheckUrl</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">Metric</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#9cdcfe;">options</span>&nbsp;<span style="color:#b4b4b4;">=&gt;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">{</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#9cdcfe;">options</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">ResultStatusCodes</span><span style="color:gainsboro;">[</span><span style="color:#b8d7a3;">HealthStatus</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">Unhealthy</span><span style="color:gainsboro;">]</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:gainsboro;">(</span><span style="color:#569cd6;">int</span><span style="color:gainsboro;">)</span><span style="color:#b8d7a3;">HttpStatusCode</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">OK</span><span style="color:gainsboro;">;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">}</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">;</span>
 
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#57a64a;">//endpoint&nbsp;health&nbsp;check&nbsp;TỔNG&nbsp;HỢP</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#9cdcfe;">app</span><span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">UseHealthChecks</span><span style="color:gainsboro;">(</span><span style="color:#9cdcfe;">healthCheckUrl</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">Health</span><span style="color:gainsboro;">,</span>&nbsp;<span style="color:#569cd6;">new</span>&nbsp;<span style="color:#4ec9b0;">HealthCheckOptions</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">{</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#57a64a;">//trả&nbsp;về&nbsp;tất&nbsp;cả&nbsp;các&nbsp;health&nbsp;checks&nbsp;(Kafka,&nbsp;DB,&nbsp;Redis...&nbsp;không&nbsp;phân&nbsp;biệt&nbsp;tag).</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">Predicate</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#9cdcfe;">_</span>&nbsp;<span style="color:#b4b4b4;">=&gt;</span>&nbsp;<span style="color:#569cd6;">true</span><span style="color:gainsboro;">,</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">ResponseWriter</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#4ec9b0;">UIResponseWriter</span><span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">WriteHealthCheckUIResponse</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:gainsboro;">}</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">;</span>
<span style="color:gainsboro;">}</span></pre>


- Gọi hàm ở `Program.cs`

<pre style="font-family:Cascadia Mono;font-size:13px;color:#dadada;background:#1e1e1e;"><span style="color:#569cd6;">var</span>&nbsp;<span style="color:#9cdcfe;">app</span>&nbsp;<span style="color:#b4b4b4;">=</span>&nbsp;<span style="color:#9cdcfe;">builder</span><span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">Build</span><span style="color:gainsboro;">(</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">;</span>
 
<span style="color:#9cdcfe;">app</span><span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">ConfigureHealthChecks</span><span style="color:gainsboro;">(</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">;</span>
 
<span style="color:#d8a0df;">if</span>&nbsp;<span style="color:gainsboro;">(</span><span style="color:#b4b4b4;">!</span><span style="color:#9cdcfe;">app</span><span style="color:#b4b4b4;">.</span><span style="color:gainsboro;">Environment</span><span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">IsProduction</span><span style="color:gainsboro;">(</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">)</span>
<span style="color:gainsboro;">{</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#9cdcfe;">app</span><span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">UseSwagger</span><span style="color:gainsboro;">(</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">;</span>
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#9cdcfe;">app</span><span style="color:#b4b4b4;">.</span><span style="color:#dcdcaa;">UseSwaggerUI</span><span style="color:gainsboro;">(</span><span style="color:gainsboro;">)</span><span style="color:gainsboro;">;</span>
<span style="color:gainsboro;">}</span></pre>
---

# Kết quả

- <host>/heath 
```json
{
  "status": "Healthy",
  "totalDuration": "00:00:01.1854012",
  "entries": {
    "live": {
      "data": {

      },
      "duration": "00:00:00.0011307",
      "status": "Healthy",
      "tags": [
        "live"
      ]
    },
    "kafka-fox": {
      "data": {

      },
      "duration": "00:00:01.0691262",
      "status": "Healthy",
      "tags": [
        "kafka-fox",
        "ready"
      ]
    },
    "account-db": {
      "data": {

      },
      "duration": "00:00:00.7459209",
      "status": "Healthy",
      "tags": [
        "account-db",
        "ready"
      ]
    },
    "trading-db": {
      "data": {

      },
      "duration": "00:00:00.7457297",
      "status": "Healthy",
      "tags": [
        "trading-db",
        "ready"
      ]
    },
    "nova-db": {
      "data": {

      },
      "duration": "00:00:00.7457321",
      "status": "Healthy",
      "tags": [
        "nova-db",
        "ready"
      ]
    },
    "main-sv-1": {
      "data": {

      },
      "duration": "00:00:01.1718701",
      "status": "Healthy",
      "tags": [
        "main-sv-1",
        "main-sv",
        "ready"
      ]
    },
    "main-sv-2": {
      "data": {

      },
      "duration": "00:00:01.1718780",
      "status": "Healthy",
      "tags": [
        "main-sv-2",
        "main-sv",
        "ready"
      ]
    }
  }
}
```
- <host>/metrics
=> Sẽ check các kết nối mà mình đăng ký
=> Hạ tầng sẽ cấu hình pull mỗi 5s => có vấn đề thì sẽ log error => alert

```
...
# TYPE healthcheck gauge
healthcheck{healthcheck="main-sv-2"} 2
healthcheck{healthcheck="kafka-fox"} 2
healthcheck{healthcheck="account-db"} 2
healthcheck{healthcheck="main-sv-1"} 2
healthcheck{healthcheck="nova-db"} 2
healthcheck{healthcheck="live"} 2
...
````