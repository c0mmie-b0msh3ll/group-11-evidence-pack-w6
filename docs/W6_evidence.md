# Week 6 Evidence Pack — TaskIO Operational Controls

## Project Recap

TaskIO là một ứng dụng quản lý công việc và workspace theo hướng Trello-style, phục vụ cho domain collaborative project operations. Người dùng có thể tạo workspace, board, column, card, task, comment, member và attachment. Ngoài phần quản lý task truyền thống, ứng dụng còn có lớp AI hỗ trợ hỏi đáp tài liệu sản phẩm và tạo tóm tắt workspace.

Kiến trúc được kế thừa từ W1-W5 là mô hình three-tier application: frontend React, backend Express/Socket.IO, MongoDB/DocumentDB làm data store chính, Redis/ElastiCache cho cache và realtime coordination, S3 cho artifact/attachment, và một lớp AI dùng Bedrock. Ở W5, nhóm đã bổ sung phần AWS networking và resilience gồm VPC peering, network segmentation, API Gateway trước Lambda, VPC Flow Logs, EFS workspace exports, AWS Backup và async Lambda failure handling.

Sang W6, nhóm không redeploy lại toàn bộ kiến trúc W1-W5 vì mục tiêu tuần này là chứng minh các operational controls. Stack demo được giữ tối giản: triển khai một AZ khi có thể, ưu tiên serverless cho AI/API path, và chỉ giữ EC2 nhỏ ở những chỗ cần evidence thật. Nhóm không dùng NAT Gateway, Network Firewall, multi-AZ expansion hoặc database replica bổ sung vì các thành phần đó không bắt buộc cho W6 must-have và sẽ làm tăng chi phí mà không giúp evidence rõ hơn.

Nhóm cũng xử lý hai feedback từ W5. Thứ nhất, evidence W5 từng làm lộ access key, nên W6 deploy flow chỉ load credential từ file CSV local lúc runtime, không hardcode key vào script hoặc tài liệu. Evidence pack cũng được scan lại trước khi nộp để tránh chứa access key/secret pattern. Thứ hai, W5 dùng nhiều hạ tầng hơn mức demo cần, nên W6 chọn hướng minimal deployment: chỉ deploy những thành phần cần để chứng minh bốn operational controls, và không thêm service mới nếu service đó không trực tiếp làm evidence mạnh hơn.

## Report Chung

Công việc tuần này được chia thành ba nhóm chính. Nam, Khanh và Hoàng phụ trách development/deployment. Tiến, Quyết và Hưng chuẩn bị presentation và slide. Thương, Nghĩa và Khôi phụ trách evidence pack và architecture diagram.

Stack W6 hiện tại đủ để demo yêu cầu của tuần này dù không phải là bản production đầy đủ của W1-W5. W6 được chấm trên operational controls: cost visibility, automated cost action, observability và self-healing security. Stack hiện có EC2, ALB, Lambda, API Gateway, CloudWatch Logs, CloudTrail, S3, SNS và EventBridge. Các thành phần này đủ để tạo evidence thật cho bốn nhóm must-have mà không cần mở rộng thêm hạ tầng tốn chi phí.

Trong lần review cost đầu tiên, nhóm cũng phát hiện một cost driver OpenSearch Serverless có sẵn trong workshop account. Finding này được đưa vào report vì nó thể hiện nhóm có điều tra khoản chi bất thường trên AWS, thay vì chỉ báo cáo các resource do nhóm chủ động deploy.

![](../week6-evidence-media/media/image5.png)

*Figure 1. Cost Explorer cho thấy khoảng 5 USD OpenSearch cost đã tồn tại trong workshop account trước khi TaskIO W6 được deploy.*

![](../week6-evidence-media/media/image6.png)

*Figure 2. CloudTrail logging evidence cho thấy account activity đang được ghi lại để phục vụ audit và điều tra.*

![](../week6-evidence-media/media/image16.png)

*Figure 3. Lần kiểm tra cost thứ hai cho thấy OpenSearch cost tăng lên gần 13 USD trong region us-west-2.*

## OpenSearch Cost Finding

Trong quá trình điều tra chi phí, nhóm phát hiện khoản charge bất thường chủ yếu đến từ một OpenSearch Serverless vector collection ở `us-west-2`. Resource này không thuộc TaskIO W6 stack. Nó nằm trong stack `rag-w-bedrock-kb` và có vẻ liên quan đến workshop RAG environment.

| Field | Value |
|---|---|
| Region | `us-west-2` |
| Service | Amazon OpenSearch Service |
| Type | OpenSearch Serverless `VECTORSEARCH` |
| Collection | `myproduct-kb` |
| Collection ID | `mgqs44w2nticsbztifo2` |
| Stack | `rag-w-bedrock-kb` |
| Initial status | `ACTIVE` |
| Standby replicas | `ENABLED` |

Cost Explorer cho thấy chi phí đến từ OpenSearch Serverless OCU usage:

```text
2026-05-19 -> 2026-05-20
USW2-IndexingOCU: 2.6453 USD
USW2-SearchOCU:   2.6450 USD
Total:            5.2903 USD

2026-05-20 -> 2026-05-21
USW2-IndexingOCU: 3.8400 USD
USW2-SearchOCU:   3.8400 USD
Total:            7.6800 USD

Observed total:   ~12.97 USD
```

Điểm đáng chú ý là OpenSearch Serverless tính phí theo OCU-hours cho cả indexing capacity và search capacity. Một vector collection vẫn có thể tạo baseline cost kể cả khi ứng dụng không query nhiều. Vì collection này không cần cho demo TaskIO W6, nhóm đã xóa collection do CloudFormation quản lý để dừng phần OCU-hour spend đang tiếp tục phát sinh.

Verification summary:

```text
CloudFormation stack: rag-w-bedrock-kb
Resource deleted:     AWS::OpenSearchServerless::Collection
Delete result:        DELETE_COMPLETE

Post-cleanup check:
aws opensearchserverless list-collections --region us-west-2
Result: []
```

Finding này củng cố hướng vận hành minimal-cost của W6: xác định cost driver, kiểm tra nó có thuộc active workload hay không, và loại bỏ nếu nó nằm ngoài phạm vi demo bắt buộc.

## MH-COST-V — Cost Visibility & Attribution

### Mục tiêu

MH-COST-V thiết lập khả năng nhìn thấy và phân bổ chi phí cơ bản cho TaskIO trên AWS. Nhóm đã triển khai tagging strategy thống nhất, verify tag trên resource đã deploy, cấu hình cost monitoring và ghi nhận billing limitation khiến workshop member account không thể tự activate custom cost allocation tags.

### Tagging Strategy

Tagging strategy của TaskIO được thiết kế cho toàn bộ full-stack application, không chỉ các resource được deploy trong W6. Theo AWS tagging best practices, nhóm tách tag theo bốn mục đích: cost allocation, automation, resource organization và security/access control. W6 chỉ deploy một subset tối thiểu để tạo evidence, nhưng cùng một tagging dictionary sẽ áp dụng cho full stack khi bật lại các thành phần W1-W5 như ALB/ASG, EFS, Backup, Network Firewall, NAT, DocumentDB, ElastiCache, S3 attachment bucket và API/Lambda layer.

Mandatory tags:

| Tag Key | Purpose | Standard Value |
|---|---|---|
| `Application` | Workload ownership and Cost Explorer grouping | `TaskIO` |
| `Environment` | Environment scope and automation filter | `dev`, `staging`, or `prod` |
| `CostCenter` | Cost allocation / showback | `G11` |
| `Owner` | Accountable team/contact | `dinhdanhnam1@gmail.com` |
| `ManagedBy` | Deployment ownership | `cloudformation`, `manual-demo`, or `terraform` |

Operational automation tags:

| Tag Key | Allowed Value | Meaning |
|---|---|---|
| `keep` | `true` | Cost Guard must not stop this resource |
| `SecurityGuardScope` | `true` | Security Guard can remediate unsafe public admin ingress |
| `CostGuardScope` | `true` | Resource is eligible for automated cost action |

Tag rules:

- Tags are case-sensitive. Use `Application=TaskIO`, not `taskIO`, `taskio`, or `Taskio`.
- Use `Environment=dev`, not `Dev` or `DEV`.
- Use one shared `Owner` contact rather than personal ad-hoc values per resource.
- Apply tags at creation time through CloudFormation/IaC when possible; use Tag Editor only for audit or repair.
- Activate `Application`, `Environment`, `CostCenter`, and `Owner` as cost allocation tags from the payer account before relying on Cost Explorer tag grouping.

Full-stack resource coverage:

| Layer | Resource Types | Required Tags | Automation Notes |
|---|---|---|---|
| Frontend/static | S3 frontend bucket, CloudFront distribution, Route 53 records | Mandatory tags | No `keep` needed; not stopped by Cost Guard |
| API edge | ALB, target groups, API Gateway REST/HTTP APIs, WAF if enabled | Mandatory tags | ALB in dev can be included in a future scale-down policy, but W6 Cost Guard only stops EC2 |
| Compute | EC2, ASG, ECS/Fargate if introduced, Lambda | Mandatory tags | Long-running backend compute uses `keep=true`; demo stop targets do not |
| Data | DocumentDB/Mongo, ElastiCache/Redis, EFS, S3 attachment bucket | Mandatory tags | Data stores should not be stopped/deleted by W6 Cost Guard; future policies need backup/snapshot checks |
| Network | VPC, subnets, route tables, NAT Gateway, Network Firewall, security groups, VPC endpoints | Mandatory tags | NAT/Firewall are cost-heavy; tag for attribution, not W6 stop automation |
| Resilience | AWS Backup vault/plans, SQS DLQ, CloudWatch alarms/log groups | Mandatory tags | Kept for audit/restore evidence; not Cost Guard stop targets |
| Operations | Cost Guard Lambda, Security Guard Lambda, EventBridge rules, SNS, Budget | Mandatory tags | These controls should normally use `keep=true` because they protect the environment |

![](../week6-evidence-media/media/image10.png)

*Figure 4. Tag Editor evidence cho backend EC2 của TaskIO với bộ tag chuẩn của Week 6.*

![](../week6-evidence-media/media/image25.png)

*Figure 5. Tag Editor evidence cho VPC của TaskIO với cùng tagging strategy đã chuẩn hóa.*

![](../week6-evidence-media/media/image.png)

*Figure 5a. Tag evidence cho Lambda `taskio-w6-summarizeWorkspace`, bổ sung resource type thứ ba ngoài EC2 và VPC.*

Các baseline resources đã được tag gồm VPC, Internet Gateway, public subnet, route table, security groups, EC2 instances, Systems Manager Parameter Store, Application Load Balancer, target group và listener. Với full-stack deployment, tagging sẽ mở rộng sang S3, CloudFront, API Gateway, Lambda, EFS, Backup, NAT Gateway, Network Firewall, DocumentDB và ElastiCache nếu các thành phần này được bật lại.

Trong workshop environment, compliance được verify thủ công qua AWS Resource Groups Tag Editor. Nếu đưa sang production, tagging nên được enforce theo hai lớp:

- Proactive: CloudFormation tags as code, AWS Organizations Tag Policies, CloudFormation Guard/hooks, và SCP guardrails cho request tạo resource thiếu tag.
- Reactive: AWS Config `required-tags`, Tag Editor audit, và Lambda remediation hoặc ticketing cho resource thiếu tag.

### Cost Allocation Tags

Nhóm đã thử activate custom cost allocation tags trong AWS Billing Console. Tuy nhiên workshop account là AWS Organizations member account, nên billing permissions bị giới hạn ở payer account level. Vì vậy nhóm không thể activate custom cost allocation tags trực tiếp từ sandbox account. Trainer đã chấp nhận limitation này sau khi nhóm giải thích rằng activation phải được thực hiện ở payer/management account, không phải từ member account dùng cho workshop. Vì tag-based Cost Explorer breakdown chưa khả dụng trong sandbox, evidence dùng service-level Cost Explorer breakdown làm fallback và vẫn giữ tagging strategy để khi payer account activate tags thì workload có thể group theo Application, Environment, CostCenter và Owner ngay lập tức.

![](../week6-evidence-media/media/image1.png)

*Figure 6. Billing console limitation cho thấy cost allocation tags phải được quản lý ở payer account level, không phải từ workshop member account.*

### Cost Monitoring Tools

AWS Cost Explorer được dùng để review service-level spend và baseline spending trend. Việc group cost theo tag chưa khả dụng vì member account không có quyền activate custom cost allocation tags, và limitation này đã được trainer chấp nhận. AWS Budgets được cấu hình theo hai lớp: monthly owner notification để theo dõi spending tổng thể, và daily cost-driven budget để nối vào SNS -> Cost Guard Lambda automation path.

| Setting | Value |
|---|---|
| Monthly Budget Name | `TaskIO-W6-Budget` |
| Monthly Budget Amount | `150 USD` |
| Monthly Period | Monthly |
| Monthly Alert Type | Actual and Forecasted |
| Notification Email | `dinhdanhnam1@gmail.com` |
| Daily Cost Guard Budget Name | `TaskIO-W6-Daily-Cost-Guard` |
| Daily Cost Guard Budget Amount | `150 USD` |
| Daily Cost Guard Period | Daily |
| Daily Cost Guard Action Path | SNS topic `healthbot-cost-guard-alerts` -> Lambda `costGuardLambda` |

![](../week6-evidence-media/media/image22.png)

*Figure 7. Cost Explorer service breakdown dùng để xác định cost driver chính trong AWS account.*

![](../week6-evidence-media/media/image4.png)

*Figure 8. Cost Explorer trend graph dùng để so sánh spending trong giai đoạn quan sát Week 6.*

![](../week6-evidence-media/media/image13.png)

*Figure 9. AWS Budget configuration cho TaskIO-W6-Budget với threshold 150 USD. Daily budget `TaskIO-W6-Daily-Cost-Guard` cũng đã được tạo để nối cost signal vào SNS -> Cost Guard path.*

![](../week6-evidence-media/media/image3.png)

*Figure 10. Cost allocation tag activation attempt bị chặn bởi billing permissions của workshop account.*

### Kết quả

MH-COST-V đã hoàn thành trong phạm vi W6. Nhóm có tagging strategy nhất quán, evidence verify resource tags, Cost Explorer evidence, Budget monitoring và evidence cho billing permission limitation.

## MH-COST-A — Cost Control & Action

MH-COST-A yêu cầu một hành động cost-control tự động, không chỉ dừng ở mức notification. Nhóm triển khai Cost Guard Lambda có khả năng stop W6 EC2 instance khi được trigger. Cách này phù hợp với minimal stack vì EC2 là compute resource rõ ràng, có thể kiểm soát được, và action có thể verify qua EC2 state cùng CloudTrail events.

Lambda sử dụng IAM role theo hướng least privilege cho EC2 stop workflow. EventBridge cung cấp scheduled execution, còn đường đi Budget to SNS to Lambda chứng minh cùng automation này có thể được nối với cost signal.

Full-stack Cost Guard policy:

| Resource Category | W6 Demo Action | Full-Stack Action When Deployed |
|---|---|---|
| Standalone dev EC2 | Stop if `Environment=dev` and no `keep=true` | Same policy; suitable for demo hosts, bastion/debug hosts, and temporary workers |
| Backend EC2 that must stay online | Add `keep=true` | Skip stop action; review manually or scale through ASG policy |
| ASG-managed backend | Not used as W6 stop target | Do not stop instances directly; scale `MinSize` and `DesiredCapacity` to `0` only for dev/non-critical ASGs with `CostGuardScope=true` |
| ECS/Fargate services | Not deployed in W6 | Future action is desired-count scale down, not task termination, and only outside production |
| NAT Gateway / Network Firewall | Not deployed in W6 | Tag for cost attribution; do not auto-delete in W6 automation because deletion changes routing/security posture |
| DocumentDB / ElastiCache / EFS | Not deployed in W6 | Excluded from W6 stop automation; future policies require snapshot/backup and explicit maintenance window |
| Lambda / API Gateway / CloudWatch / SNS / EventBridge | Kept running | Do not stop; these are low-cost controls or observability surfaces |

For the current account, the intended `keep=true` resource is the running backend instance `taskio-w6-backend-single-ec2`. The demo instance remains without `keep=true` so Cost Guard can produce clear `StopInstances` evidence. This keeps the W6 evidence simple while documenting how the same tag model would protect or control full-stack resources later.

![](../week6-evidence-media/media/image2.png)

*Figure 11. Cost Guard Lambda IAM role evidence cho least-privilege permissions dùng trong EC2 stop automation.*

![](../week6-evidence-media/media/image17.png)

*Figure 12. Manual Cost Guard Lambda test trả về success và trigger EC2 stop path.*

![](../week6-evidence-media/media/image31.png)

*Figure 13. EC2 console evidence cho thấy target instance chuyển sang stopping/stopped sau automation.*

![](../week6-evidence-media/media/image32.png)

*Figure 14. EventBridge daily schedule được cấu hình để tự động invoke Cost Guard Lambda.*

![](../week6-evidence-media/media/image12.png)

*Figure 15. CloudTrail event evidence cho EC2 stop action được tạo bởi Cost Guard workflow.*

![](../week6-evidence-media/media/image27.png)

*Figure 16. EC2 instance state evidence xác nhận Week 6 demo instance đã được stop bởi control.*

![](../week6-evidence-media/media/image8.png)

*Figure 17. AWS Budget to SNS to Lambda cost-driven path được cấu hình cho automated cost response.*

![](../week6-evidence-media/media/image20.png)

*Figure 18. SNS topic/subscription evidence cho notification path kết nối đến Cost Guard Lambda.*

![](../week6-evidence-media/media/image23.png)

*Figure 19. SNS publish test evidence cho thấy cost notification path có thể trigger downstream automation.*

![](../week6-evidence-media/media/image15.png)

*Figure 20. Final Cost Guard integration evidence cho Budget, SNS và Lambda response flow.*

### Cost Latency ADR

AWS cost data is not real-time. Cost Explorer and AWS Budgets can lag by about 8-24 hours, so a newly created daily budget may not cross or publish a real cost threshold during a short workshop demo window. For production, TaskIO keeps the real daily budget connected to SNS and Cost Guard, then uses a manual SNS publish during the demo to prove the exact same downstream action path. This avoids waiting for delayed billing data while still demonstrating the operational response: cost signal -> SNS -> Cost Guard Lambda -> EC2 stop action -> CloudTrail evidence.

Production behavior is intentionally conservative: the W6 Cost Guard stops only standalone non-production compute that is tagged `Environment=dev`, eligible for cost action, and not protected by `keep=true`. For ASG/ECS/database resources, production policy should use scale-down, snapshot/backup, or maintenance-window controls instead of destructive stop/delete behavior.

## MH-OBS — Observability

Phạm vi observability của TaskIO W6 tập trung vào AI/API path đã redeploy, vì đây là user-facing operation có giá trị nhất trong minimal stack. Dashboard kết hợp application-level telemetry, managed service metrics và host-level metrics để nhóm kiểm tra được AI path có reachable không, Lambda có healthy không, và demo host có basic disk telemetry không.

Custom metric bắt buộc được emit từ application code, không phải push thủ công bằng CLI. Lambda `askDocsBot` publish `AskDocsLatencyMs` và `AskDocsErrorCount` vào namespace `TaskIO/Operations`. Lambda `summarizeWorkspace` cũng publish `WorkspaceSummaryLatencyMs` và `WorkspaceSummaryErrorCount` khi summary path chạy hoặc gặp lỗi. Các metric hiện đang dùng dimensions `Application=TaskIO`, `Environment=dev` và `Function=askDocsBot` hoặc `Function=summarizeWorkspace`, giúp dashboard được scope đúng vào TaskIO workload.

API Gateway observability được thêm như một phần enrichment chi phí thấp. W6 REST API `taskio-w6-ai-rest-api` đã bật stage-level access logging vào `/aws/rest-api/taskio-w6-ai-rest-api`, bật method metrics, và có saved Logs Insights query để aggregate requests theo status, method và path. Để tạo datapoints cho dashboard, nhóm đã gọi `OPTIONS /ai/docs/ask` và `OPTIONS /ai/workspaces/{workspaceId}/summary` nhiều lần; access logs ghi nhận status `200` cho cả hai route.

Saved Logs Insights query evidence:

| Field | Value |
|---|---|
| Saved query name | `W6-TaskIO-ApiGateway-Requests-By-Status` |
| Saved query name | `W6-TaskIO-ApiGateway-Recent-Raw-Rows` |
| Log group | `/aws/rest-api/taskio-w6-ai-rest-api` |
| Query purpose | Aggregate API Gateway requests by status/method/path and show recent raw request rows |

Saved query text for recent rows:

```sql
fields @timestamp, @message
| sort @timestamp desc
| limit 20
```

Observed Logs Insights result rows from `/aws/rest-api/taskio-w6-ai-rest-api`:

| Timestamp UTC | Method | Path | Status |
|---|---|---|---|
| `2026-05-22 01:44:49` | POST | `/ai/workspaces/{workspaceId}/summary` | 500 |
| `2026-05-22 01:44:48` | POST | `/ai/workspaces/{workspaceId}/summary` | 500 |
| `2026-05-22 01:44:47` | POST | `/ai/workspaces/{workspaceId}/summary` | 500 |
| `2026-05-22 01:44:46` | POST | `/ai/workspaces/{workspaceId}/summary` | 500 |
| `2026-05-22 01:44:45` | POST | `/ai/workspaces/{workspaceId}/summary` | 500 |
| `2026-05-22 01:44:44` | POST | `/ai/workspaces/{workspaceId}/summary` | 500 |

This satisfies the saved-query requirement with a named query, explicit log group, and more than five visible result rows from real API Gateway access logs.

| Layer | Signal | Source |
|---|---|---|
| Application | `TaskIO/Operations / AskDocsLatencyMs` | `askDocsBot` Lambda code qua `PutMetricData` |
| Application | `TaskIO/Operations / AskDocsErrorCount` | `askDocsBot` Lambda error path |
| Application | `TaskIO/Operations / WorkspaceSummaryLatencyMs` | `summarizeWorkspace` Lambda code qua `PutMetricData` |
| Application | `TaskIO/Operations / WorkspaceSummaryErrorCount` | `summarizeWorkspace` Lambda error path |
| Lambda runtime | `Errors` | AWS/Lambda standard metric |
| Lambda runtime | `Duration` | AWS/Lambda standard metric |
| API edge | `Count` | AWS/ApiGateway standard metric cho `taskio-w6-ai-rest-api` |
| Host | `disk_used_percent` | CloudWatch Agent `CWAgent` namespace |

Custom metric implementation:

```js
await cloudWatchClient.send(new PutMetricDataCommand({
  Namespace: 'TaskIO/Operations',
  MetricData: [{
    MetricName: 'AskDocsLatencyMs',
    Value: Date.now() - startedAt,
    Unit: 'Milliseconds',
    Dimensions: [
      { Name: 'Application', Value: 'TaskIO' },
      { Name: 'Environment', Value: 'dev' },
      { Name: 'Function', Value: 'askDocsBot' }
    ]
  }]
}))
```

Lambda execution role chỉ cho phép `cloudwatch:PutMetricData` trong namespace `TaskIO/Operations`. Cách này giữ permission đúng phạm vi custom metric namespace dùng cho W6 dashboard.

Sau khi invoke để tạo dữ liệu demo, CloudWatch đã có datapoints cho cả hai Lambda:

```text
AskDocsLatencyMs:              5 samples, average ~116.8 ms
AskDocsErrorCount:             sum 5
WorkspaceSummaryLatencyMs:     5 samples, average ~150 ms
WorkspaceSummaryErrorCount:    sum 5
```

Lưu ý: tagging strategy chuẩn hóa Application=TaskIO. Custom metric dimension trong CloudWatch cũng dùng Application=TaskIO, nên dashboard và evidence nên chọn cùng dimension này để tránh lệch scope.

![](../week6-evidence-media/media/image29.png)

*Figure 21. CloudWatch dashboard overview cho MH3, gom các TaskIO observability widgets vào một màn hình.*

![](../week6-evidence-media/media/image33.png)

*Figure 22. CloudWatch dashboard widget cho API Gateway request metrics của TaskIO AI REST API.*

![](../week6-evidence-media/media/image14.png)

*Figure 23. CloudWatch dashboard widget cho custom Lambda application metrics được publish vào TaskIO/Operations.*

![](../week6-evidence-media/media/image30.png)

*Figure 24. CloudWatch alarm configuration cho Week 6 monitoring control.*

![](../week6-evidence-media/media/image28.png)

*Figure 25. CloudWatch alarm state evidence cho thấy alarm có thể được evaluate từ dashboard.*

![](../week6-evidence-media/media/image21.png)

*Figure 26. CloudWatch Logs Insights query result dùng để phân tích API Gateway access logs theo method, path và status.*

## MH-SEC — Self-Healing Security

Self-healing security control của TaskIO W6 bảo vệ hệ thống khỏi accidental public administrative access. Misconfiguration được chọn là Security Group ingress rule mở SSH ra internet. Đây là failure mode thực tế vì rule debug tạm thời rất dễ bị để quên sau khi troubleshoot.

Blast radius if not remediated: a public SSH/RDP rule exposes the backend administration surface to internet-wide scanning and brute-force attempts. If an attacker compromises the host, they can pivot from the EC2 instance into application secrets, logs, S3 access paths, and any database/cache network path allowed from that security group. Even if the application itself is healthy, one forgotten admin ingress rule can turn a temporary troubleshooting change into a broader workload compromise. The self-healing guard reduces this blast radius by removing the risky ingress rule shortly after CloudTrail records the change.

Detect-and-fix loop được thiết kế theo event-driven pattern. CloudTrail ghi lại unsafe `AuthorizeSecurityGroupIngress` API call, EventBridge match event, và Security Guard Lambda gọi `RevokeSecurityGroupIngress` để xóa public rule.

Full-stack Security Guard scope:

| Resource Category | Guard Behavior |
|---|---|
| W6 test Security Group | Remediate public SSH/RDP ingress immediately for demo evidence |
| Backend/API Security Groups | Remediate `0.0.0.0/0` on admin ports `22` and `3389`; leave intended app ports such as `80`/`443` to ALB/API edge design |
| ALB/WAF edge | Do not remove public `80`/`443` rules automatically; inspect through WAF/ALB policy instead |
| Database/cache Security Groups | Remediate any public ingress; these should only allow app/security-group sources |
| EFS/Backup/S3 | Security Guard does not stop resources; S3 exposure is handled by Block Public Access and bucket policies |
| Resources not deployed in W6 | Same policy applies once deployed if tagged `Application=TaskIO`; automation can be narrowed with `SecurityGuardScope=true` |

For W6, only the test Security Group is intentionally violated. For the full stack, the same self-healing rule would protect backend, database, cache, and EFS-facing security groups from accidental public admin exposure while avoiding disruption to legitimate public edge resources.

```text
Violation:
0.0.0.0/0 on TCP port 22

Remediation:
ec2:RevokeSecurityGroupIngress
```

Demo loop:

1. Demo Security Group được cố ý mở `0.0.0.0/0` trên TCP port `22`.
2. EventBridge trigger Security Guard Lambda từ CloudTrail `AuthorizeSecurityGroupIngress` event.
3. Lambda gọi `RevokeSecurityGroupIngress`.
4. Public SSH rule bị xóa khỏi Security Group.
5. CloudTrail ghi lại `RevokeSecurityGroupIngress` remediation event.

Supporting preventive control là S3 public/data exposure prevention. S3 account-level Block Public Access đã được bật, và W6 artifact bucket có deny policy để reject unencrypted object uploads. Unencrypted `PutObject` test fail với `AccessDenied`, trong khi encrypted `PutObject` test thành công với `AES256` server-side encryption.

Security Guard Lambda và EventBridge rule gần như không tạo chi phí ở workshop traffic volume vì chỉ chạy khi có security event. S3 Block Public Access và deny-unencrypted bucket policy là free controls, nên chúng cải thiện security baseline mà không làm tăng rủi ro vượt `150 USD` weekly cost cap.

![](../week6-evidence-media/media/image18.png)

*Figure 27. MH4 self-healing setup evidence cho EventBridge và Lambda security automation path.*

![](../week6-evidence-media/media/image24.png)

*Figure 28. Security group violation trước remediation, dùng để demo self-healing security loop.*

![](../week6-evidence-media/media/image26.png)

*Figure 29. Self-healing result sau khi security automation xóa risky inbound rule.*

![](../week6-evidence-media/media/image19.png)

*Figure 30. Security Guard Lambda code/configuration evidence cho remediation logic.*

![](../week6-evidence-media/media/image11.png)

*Figure 31. S3 account-level Block Public Access preventive control được bật như supporting security guardrail.*

![](../week6-evidence-media/media/image9.png)

*Figure 32. S3 bucket policy preventive control deny uploads không dùng server-side encryption.*

![](../week6-evidence-media/media/image7.png)

*Figure 33. S3 upload test evidence cho thấy preventive control block non-compliant object uploads.*

## Bonus Stretch Goals

### Bonus 1 — Wasteful → Changed Reflection

Nhóm phát hiện khoản lãng phí lớn nhất không đến từ TaskIO W6 stack mà đến từ một OpenSearch Serverless vector collection `myproduct-kb` trong region `us-west-2`, thuộc stack `rag-w-bedrock-kb`. Cost Explorer cho thấy resource này tạo khoảng `12.97 USD` trong hai ngày qua `USW2-IndexingOCU` và `USW2-SearchOCU`, dù nó không phục vụ demo TaskIO. Nhóm đã kiểm tra resource ownership, xác nhận collection không thuộc các stack `taskio-w6-*`, sau đó xóa CloudFormation-managed collection. Sau cleanup, `aws opensearchserverless list-collections --region us-west-2` trả về danh sách rỗng. Thay đổi này loại bỏ cost driver lớn nhất, củng cố chiến lược W6 minimal-cost, và giúp account không tiếp tục bị charge OCU-hour ngoài phạm vi bài nộp.

### Bonus 2 — RI / Savings Plans Break-Even Analysis

Compute footprint W6 hiện tại chỉ có hai EC2 `t3.micro` ở `us-east-1a`: một backend EC2 đang chạy và một demo EC2 đang stopped. Hai EBS volumes hiện đều là `gp3` với `8 GiB`, `3000 IOPS` và `125 MiB/s`, nên nhóm không chọn gp2→gp3 migration làm bonus vì không còn gp2 volume thật để migrate.

Pricing API cho `Linux t3.micro` ở `US East (N. Virginia)` trả về On-Demand price là `0.0104 USD/hour`. Nếu cả hai instance chạy liên tục trong một tuần, EC2 compute cost chỉ khoảng:

```text
2 instances * 24 hours/day * 7 days * 0.0104 USD/hour = 3.49 USD/week
```

Một 1-year Standard Reserved Instance all-upfront cho `t3.micro` có upfront fee khoảng `53 USD`. Break-even so với On-Demand nếu workload chạy liên tục là:

```text
53 USD / 0.0104 USD/hour = ~5,096 hours
5,096 hours / 24 = ~212 days
```

Vì W6 workshop workload chỉ tồn tại khoảng một tuần và compute spend dự kiến thấp hơn rất nhiều so với mức cam kết 1 năm, nhóm quyết định không mua RI hoặc Savings Plan. Quyết định hợp lý hơn cho W6 là dùng On-Demand nhỏ nhất có thể, stop demo EC2 khi không cần, giữ backend ở `t3.micro`, và không thêm managed service đắt tiền. Nếu TaskIO chuyển sang production với workload chạy ổn định trên cùng footprint trong ít nhất 7 tháng, lúc đó RI hoặc Compute Savings Plan mới đáng được xem xét.
