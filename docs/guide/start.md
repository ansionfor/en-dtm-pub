# Introduction

## What is DTM?

DTM is the first Golang distributed transaction management framework. What is different from other frameworks is DTM provides some extremely simple access methods(HTTP/GRPC), and it support multiple languages, what is more it deal with all kinds of sub-transaction disorder problems at the framework layer.

You can learn more about the original intention of DTM here([Why DTM](./why)).

## Advantage

* Easy to access
  - Support HTTP, GRPC, provides a very simple interface, greatly reduce the difficulty of getting started with distributed transaction, novices can quickly access.
* Easy to use
  - Developers no longer need to worry about suspension, empty compensation, idempotence etc., the frame has been processed
* Multiple languages
  - Suitable for companies with multi-language stacks. Such as go, python, php, nodejs, ruby.
* Easy to deploy and expand
  - Rely only on mysql, simple deployment, easy clustering, easy horizontal expansion.
* Multiple distributed transaction protocol support
  - TCC, SAGA, XA, Message

Invited to participate in the China Database Conference to share [Practice of Distributed Transaction in Multilingual Environment](http://dtcc.it168.com/yicheng.html#b9)

## Who is using DTM

[Ivydad](https://ivydad.com)

[Eglass](https://epeijing.cn)

<a style="
    background-color:#646cff;
    font-size: 0.9em;
    color: #fff;
    margin: 0.2em 0;
    width: 200px;
    text-align: center;
    padding: 12px 24px;
    display: inline-block;
    vertical-align: middle;
    border-radius: 2em;
    font-weight: 600;
" href="../other/opensource">Contrast with seata</a>

## Start

::: tip Basic knowledge
This tutorial assumes that you already have the basic knowledge of distributed transactions. If you are not familiar with this aspect, you can read [Distributed Transaction Theory](../guide/theory)

This tutorial also assumes that you have a certain programming foundation and can roughly understand the code of the Golang, if you are not familiar with this aspect, visit [Golang](https://golang.google.cn/)
:::

- [Install](./install) by the way of the recommended method of Golang.

The easiest way to try DTM is to use QuickStart demo, the main file of this example is in [dtm/example/quick_start.go](https://github.com/yedf/dtm/blob/main/examples/quick_start.go).

You can run this example with the following command in the dtm directory:

`go run app/main.go quick_start`

In this example, a saga distributed transaction is created and then submitted to dtm. The core code is as follows:

``` go
	req := &gin.H{"amount": 30} // params
	// DtmServer is the address of DTM service
	saga := dtmcli.NewSaga(DtmServer, dtmcli.MustGenGid(DtmServer)).
		// Add a TransOut sub-transaction, Forward operation is url: qsBusi+"/TransOut",  The reverse operation is url: qsBusi+"/TransOutCompensate"
		Add(qsBusi+"/TransOut", qsBusi+"/TransOutCompensate", req).
		// Add a TransIn sub-transaction, Forward operation is url: qsBusi+"/TransIn",  The reverse operation is url: qsBusi+"/TransInCompensate"
		Add(qsBusi+"/TransIn", qsBusi+"/TransInCompensate", req)
	// Submit the saga transaction, dtm will complete all sub-transactions or roll back all sub-transactions
	err := saga.Submit()
```

In this distributed transaction, a scenario in a cross-bank transfer distributed transaction is simulated. The global transaction includes TransOut (transfer out sub-transaction) and TransIn (transfer in sub-transaction), and each sub-transaction includes forward operation and reverse compensation. Definition As follows:

``` go
func qsAdjustBalance(uid int, amount int) (interface{}, error) {
	err := dbGet().Transaction(func(tx *gorm.DB) error {
		return tx.Model(&UserAccount{}).Where("user_id = ?", uid).Update("balance", gorm.Expr("balance + ?", amount)).Error
	})
	if err != nil {
		return nil, err
	}
	return M{"dtm_result": "SUCCESS"}, nil
}

func qsAddRoute(app *gin.Engine) {
	app.POST(qsBusiAPI+"/TransIn", common.WrapHandler(func(c *gin.Context) (interface{}, error) {
		return qsAdjustBalance(2, 30)
	}))
	app.POST(qsBusiAPI+"/TransInCompensate", common.WrapHandler(func(c *gin.Context) (interface{}, error) {
		return qsAdjustBalance(2, -30)
	}))
	app.POST(qsBusiAPI+"/TransOut", common.WrapHandler(func(c *gin.Context) (interface{}, error) {
		return qsAdjustBalance(1, -30)
	}))
	app.POST(qsBusiAPI+"/TransOutCompensate", common.WrapHandler(func(c *gin.Context) (interface{}, error) {
		return qsAdjustBalance(1, 30)
	}))
}
```

The whole transaction is finally completed successfully, the timing diagram is as follows:

![saga_normal](../imgs/saga_normal.jpg)

In actual business, sub-transactions may fail. For example, the transferred sub-account is frozen and the transfer fails. Modify the business code to make the forward operation of TransIn fail, and then look at the result.

``` go
	app.POST(qsBusiAPI+"/TransIn", common.WrapHandler(func(c *gin.Context) (interface{}, error) {
		return M{"dtm_result": "FAILURE"}, nil
	}))
```

Run this example again, the whole transaction finally fails, the sequence diagram is as follows:

![saga_rollback](../imgs/saga_rollback.jpg)

In the case of a failed transfer operation, the compensation branch of TransIn and TransOut has been executed, ensure that the final balance is the same as before the transfer.

## Are you ready?

We just introduced a complete distributed transaction in a simple way, including a successful one and a rollback. Now you should have a concrete understanding of distributed transactions. This tutorial will take you step by step to learn the technical solutions and techniques for dealing with distributed transactions.

## Chat group

Please add yedf2008(wechat) as a friend, verification reply "dtm" follow the instructions to introduce the group.

![yedf2008](http://service.ivydad.com/cover/dubbingb6b5e2c0-2d2a-cd59-f7c5-c6b90aceb6f1.jpeg)

If you think [dtm](https://github.com/yedf/dtm) is not bad or it is helpful to you, please give me a star!
