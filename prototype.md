## 原型模式作业

* 运用原型模式重构一段业务代码。

###

* 重构前：

```
public ProcessResult requestForPayment(CoobillH5PayRequest coobillH5PayRequest) {
    
    // execute payment
    CoobillH5PayParam payParam = new CoobillH5PayParam();
    payParam.setTransId(coobillH5PayRequest.getTransId());
    payParam.setOrderId(coobillH5PayRequest.getOrderId());
    payParam.setPartnerID(coobillH5PayRequest.getPartnerID());
    payParam.setPartnerName(coobillH5PayRequest.getPartnerName());
    payParam.setPayAmount(coobillH5PayRequest.getPayAmount());
    payParam.setPayMethod(coobillH5PayRequest.getPayMethod());
    payParam.setCurrencyType(coobillH5PayRequest.getCurrencyType());
    payParam.setSignInfo(coobillH5PayRequest.getSignInfo());

    payParam.setPayeeName("");
    payParam.setPayeeSubsId("");
    payParam.setPayeeTelNo("");

    payParam.setPayerName("");
    payParam.setPayerSubsId("");
    payParam.setPayerTelNo("");

    payParam.setCtxInfo(null);

    // pre-processing
    ProcessResult processResult=this.preProcessForPayment(payParam);

    ...
}
```

* 重构后：

```
public ProcessResult requestForPayment(CoobillH5PayRequest coobillH5PayRequest) {
    
    // execute payment
    CoobillH5PayParam payParam = coobillH5PayRequest.clone();

    payParam.setPayeeName("xxx");
    payParam.setPayeeSubsId("xxx");
    payParam.setPayeeTelNo("xxx");

    payParam.setPayerName("xxx");
    payParam.setPayerSubsId("xxx");
    payParam.setPayerTelNo("xxx");

    payParam.setCtxInfo(null);

    // pre-processing
    ProcessResult processResult=this.preProcessForPayment(payParam);

    ...
}
```
