还是从[Martin Fowler](https://martinfowler.com/eaaDev/AuditLog.html)开始。

> Audit log是最简单，同时也是最有效的操作记录追踪方法。其核心思想是，任何操作发生的时候都保留一条记录，记下当时的时间及发生了什么事。 (2004年3月7日)

#### 如何记录审计日志

>管理审计的目的是记录谁在什么时间做过什么事。

审计日志也是日志，和普通的日志记录方法没有区别。

#### 如何保存审计日志

##### 1. [Oracle关于审计数据库的设计](https://docs.oracle.com/cd/E19944-01/819-4483/audit_log_reference.html)

| Argument | Type | Description |
| --- | --- | --- |
| type | String | Name of the object type that is being audited |
| action | String | Name of the action that was performed |
| status | String | Name of the status for the specified action |
| name | String | Name of the object being affected by the specified action |
| resource | String | (Optional) Name of the resource where the object being changed resides |
| accountId | String | (Optional) Account ID that is being modified. This should be a native resource account name |
| error | String | (Optional) Localized error string to accompany any failures. |
| reason | String | (Optional) Name of the ReasonDenied object, which maps to an internationalized message describing the causes of common failures. |
| attributes | Map | (Optional) Map of attribute names and values that were added or modified |
| parameters | Map | (Optional) Maps up to five additional names or values that are relevant to an event |
| organizations | List | List of organization names or IDs where this event will be placed. |
| originalAttributes | Map | (Optional) Map of old attribute values |

##### 2. [Google Cloud的审计日志](https://cloud.google.com/logging/docs/audit/)
1. 审计日志类型：
- 管理员活动 activity
- 系统事件 system_event
- 数据访问 data_access

2. AuditLog JSON representation
```
{
  "serviceName": string,           # "datastore.googleapis.com"
  "methodName": string,         # "google.datastore.v1.Datastore.RunQuery"
  "resourceName": string,        # "shelves/SHELF_ID/books"
  "numResponseItems": string,
  "status": {
    object(Status)                       # google.rpc.Code
  },
  "authenticationInfo": {
    object(AuthenticationInfo)
  },
  "authorizationInfo": [
    {
      object(AuthorizationInfo)
    }
  ],
  "requestMetadata": {
    object(RequestMetadata)
  },
  "request": {
    object
  },
  "response": {
    object
  },
  "serviceData": {
    "@type": string,
    field1: ...,
    ...
  },
}
```

3. AuthenticationInfo
```
{
  "principalEmail": string,
  "authoritySelector": string,
}
```

4. AuthorizationInfo
```
{
  "resource": string,
  "permission": string,
  "granted": boolean,
}
```

5. RequestMetadata
```
{
  "callerIp": string,
  "callerSuppliedUserAgent": string,
}
```

##### 3. [Microsoft Azure的审计日志](https://docs.microsoft.com/en-us/azure/security/azure-log-audit)
Types of logs in Azure:
- Control/management logs
- Data plane logs
- Processed events

##### 4. [Amazon AWS的审计日志](https://docs.aws.amazon.com/zh_cn/organizations/latest/userguide/orgs_monitoring.html)

一段在调用API时生成的CloudTrail日志条目：
```
{
   "eventVersion":"1.06",
   "userIdentity":{
      "type":"IAMUser",
      "principalId":"AIDAMVNPBQA3EXAMPLE",
      "arn":"arn:aws:iam::111122223333:user/diego",
      "accountId":"111122223333",
      "accessKeyId":"AKIAIOSFODNN7EXAMPLE",
      "userName":"diego"
   },
   "eventTime":"2018-06-21T22:06:27Z",
   "eventSource":"organizations.amazonaws.com",
   "eventName":"CreateAccount",
   "awsRegion":"us-east-1",
   "sourceIPAddress":"192.168.0.1",
   "userAgent":"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/55.0.2883.95 Safari/537.36",
   "requestParameters":{
      "email":"anaya@amazon.com",
      "accountName":"****"
   },
   "responseElements":{
      "createAccountStatus":{
         "accountName":"****",
         "state":"IN_PROGRESS",
         "id":"car-examplecreateaccountrequestid111",
         "requestedTimestamp":"Jun 21, 2018 10:06:27 PM"
      }
   },
   "requestID":"EXAMPLE8-90ab-cdef-fedc-ba987EXAMPLE",
   "eventID":"EXAMPLE8-90ab-cdef-fedc-ba987EXAMPLE",
   "eventType":"AwsApiCall",
   "recipientAccountId":"111111111111"
}
```

##### 5. [IBM Cloud的审计日志](https://console.bluemix.net/docs/services/Cloudant/offerings/audit.html#-)
审计日志记录的信息
- 主体：账户凭证、API密钥或IBM Cloud IAM凭证
- 操作：所执行的操作，如文档读取
- 资源：有关所访问的账户、数据库和文档或所运行的查询的详细信息
- 时间：对事件时间和日期的记录

##### 6. [阿里云的操作审计](https://help.aliyun.com/document_detail/28804.html)
Action Trail

操作事件(Event)结构定义

| 字段 | 类型 | 描述 |
| --- | --- | --- |
| apiVersion | String | 所调用的云服务API版本 |
| eventId | String | 事件ID，由ActionTrail服务为每个操作事件所产生的一个GUI |
| eventName | String | API操作名称，可参考各服务的API操作列表，比如ECS的StopInstance |
| eventSource | String | 处理API请求的服务端，比如ram.aliyuncs.com |
| eventTime | String | UTC格式的时间 |
| eventType | String | 发生的事件类型，比如ApiCall或ConsoleSignin |
| eventVersion | String | ActionTrail事件格式的版本，当前版本为1 |
| errorCode | String | (Optional) 如果云服务处理API请求时发生了错误，这里记录错误码，比如NoPermission |
| errorMessage | String | (Optional) 错误消息，比如 You are not authorized |
| requestId | String | 云服务处理API请求时所产生的消息请求ID |
| requestParameters | Map | (Optional) API请求的输入参数，具体参数含义需参考相应云服务的API文档 |
| responseElements | Map | (Optional) API响应的数据，具体格式需参考相应云服务的API文档 |
| referencedResources | Map | (Optional) API操作的资源 |
| serviceName | String | 云服务名称，如Ecs, Rds, Ram |
| sourceIpAddress | String | 发送 API 请求的源 IP 地址。如果 API 请求是由用户通过控制台操作触发，那么这里记录的是用户浏览器端的 IP 地址，而不是控制台 Web 服务器的IP地址 |
| userAgent | String | 发送 API 请求的客户端代理标识，比如控制台为AliyunConsole，SDK 为aliyuncli/2.0.6 |
| userIdentity | Map | 请求者的身份信息 |

userIdentity包含的字段

| 名称 | 是否必须 | 描述 |
| --- | --- | --- |
| type | 是 | 身份类型，比如root-account, ram-user, assumed-role |
| principalId | 是 | 当前请求者的ID，如果身份类型是root-account则记录主账号ID，如果是ram-user则记录RAM用户名，如果是assumed-role则记录RoleID:RoleSessionName |
| accountId | 是 | 主账号ID |
| accessKeyId | 否 | 请求者通过SDK访问时记录 |
| userName | 否 | 如果请求者类型为ram-user则记录RAM用户名，如果是assumed-role则记录RoleID:RoleSessionName |
| sessionContext | 否 | 如果调用 API 请求时使用的是临时安全凭证 配置STS Token，则记录该信息；通过控制台执行操作时也会触发session的创建并记录该信息。Session 内容包括：creationDate（Session创建时间）、mfaAuthenticated（用户登录控制台时是否使用多因素认证） |

示例：
```
"userIdentity": {
    "type": "ram-user",
    "principalId": "28815334868278****",
    "accountId": "112233445566****",
    "accessKeyId": "55nCtAwmPLkk****",
    "userName": "B**"
}
```