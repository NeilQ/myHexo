title: web api 对[serialiable]特性的对象序列化问题
date: 2015-08-15 14:01:23
tags:
- .NET
- web api
categories:
- .NET
---

web api 2 中, 假如返回对象加了[serializable]特性，返回的json、xml数据都是包含对象的私有变量，而非共有属性，如：
```json
{
  "Total": 1,
  "Data": [
    {
      "_ticketnumber": "sample string 1",
      "_onlinedate": "sample string 2",
      "_name": "sample string 3",
      "_price": "sample string 4",
      "_verificationtime": "sample string 5",
      "_type": "sample string 6",
      "_authmethod": "sample string 7",
      "_authaccount": "sample string 8",
      "_authstores": "sample string 9",
      "_source": "sample string 10",
      "_uniqueid": "sample string 11",
      "_createtime": "2015-08-15T14:14:40.8733315+08:00"
    }
  ]
}
```

```xml
<PageableModelOfOrderYGLG4_PHE xmlns:i="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://schemas.datacontract.org/2004/07/xxx.Api.DTOs">
  <Data xmlns:d2p1="http://schemas.datacontract.org/2004/07/xxx.Model">
    <d2p1:Order>
      <d2p1:_authaccount>sample string 8</d2p1:_authaccount>
      <d2p1:_authmethod>sample string 7</d2p1:_authmethod>
      <d2p1:_authstores>sample string 9</d2p1:_authstores>
      <d2p1:_createtime>2015-08-15T14:14:40.8733315+08:00</d2p1:_createtime>
      <d2p1:_name>sample string 3</d2p1:_name>
      <d2p1:_onlinedate>sample string 2</d2p1:_onlinedate>
      <d2p1:_price>sample string 4</d2p1:_price>
      <d2p1:_source>sample string 10</d2p1:_source>
      <d2p1:_ticketnumber>sample string 1</d2p1:_ticketnumber>
      <d2p1:_type>sample string 6</d2p1:_type>
      <d2p1:_uniqueid>sample string 11</d2p1:_uniqueid>
      <d2p1:_verificationtime>sample string 5</d2p1:_verificationtime>
    </d2p1:Order>
  </Data>
  <Total>1</Total>
</PageableModelOfOrderYGLG4_PHE>
```

这是因为加了[serialiable]特性的对象将会被序列化出所有的字段，包括private与public的，
要解决这个问题，我们可以移除这个特性，或者在WebApiConfig.cs中加入如下代码，以修改
序列化的方式：

```csharp
config.Formatters.JsonFormatter.SerializerSettings.ContractResolver = new DefaultContractResolver();
config.Formatters.XmlFormatter.UseXmlSerializer = true;
```

如此，返回的json或xml包含的就都是共有属性了，同时我们也能发现，xml数据中针对不同对象
的命名空间属性xmlns也去掉了。
```json
{
  "Total": 1,
  "Data": [
    {
      "TicketNumber": "sample string 1",
      "OnlineDate": "sample string 2",
      "Name": "sample string 3",
      "Price": "sample string 4",
      "VerificationTime": "sample string 5",
      "Type": "sample string 6",
      "AuthMethod": "sample string 7",
      "AuthAccount": "sample string 8",
      "AuthStores": "sample string 9",
      "Source": "sample string 10",
      "UniqueID": "sample string 11",
      "CreateTime": "2015-08-15T14:17:05.1900932+08:00"
    }
  ]
}
```

```xml
<PageableModelOfOrder xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <Total>1</Total>
  <Data>
    <Order>
      <TicketNumber>sample string 1</TicketNumber>
      <OnlineDate>sample string 2</OnlineDate>
      <Name>sample string 3</Name>
      <Price>sample string 4</Price>
      <VerificationTime>sample string 5</VerificationTime>
      <Type>sample string 6</Type>
      <AuthMethod>sample string 7</AuthMethod>
      <AuthAccount>sample string 8</AuthAccount>
      <AuthStores>sample string 9</AuthStores>
      <Source>sample string 10</Source>
      <UniqueID>sample string 11</UniqueID>
      <CreateTime>2015-08-15T14:17:05.1900932+08:00</CreateTime>
    </Order>
  </Data>
</PageableModelOfOrder>
```

