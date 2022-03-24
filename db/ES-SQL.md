```
{
  "settings": {
    "index": {
      "number_of_shards": "2",
      "number_of_replicas": "1"
    },
    "analysis": {
      "analyzer": {
        "order_analyzer": {
          "pattern": "[,|]",
          "type": "pattern"
        }
      }
    }
  },
  "mappings": {
    "order": {
      "_routing": {
        "required": true
      },
      "properties": {
        "dispathId": {
          "type": "integer"
        },
        "orderType": {
          "type": "short"
        },
        "customsModelDyS": {
          "type": "keyword"
        },
        "orderId": {
          "type": "long"
        },
        "venderIdFusionType": {
          "type": "keyword"
        },
        "logisticsExt": {
          "index": false,
          "type": "keyword"
        },
        "county": {
          "type": "integer"
        },
        "ljDtDyTdt": {
          "type": "date"
        },
        "paymentConfirmTime": {
          "type": "date"
        },
        "state2": {
          "index": false,
          "type": "short"
        },
        "taxFeeDyL": {
          "type": "integer"
        },
        "dparentId": {
          "type": "long"
        },
        "isPresaleOrder": {
          "type": "integer"
        },
        "customsIdDyS": {
          "type": "keyword"
        },
        "commissionDyL": {
          "index": false,
          "type": "long"
        },
        "province": {
          "type": "integer"
        },
        "recommendDeliveryIdDyL": {
          "type": "long"
        },
        "yn": {
          "type": "short"
        },
        "couponEventSum": {
          "type": "long"
        },
        "erpSynchTime": {
          "index": false,
          "type": "date"
        },
        "id": {
          "type": "text",
          "fields": {
            "keyword": {
              "ignore_above": 256,
              "type": "keyword"
            }
          }
        },
        "state": {
          "index": false,
          "type": "short"
        },
        "dispathName": {
          "type": "keyword"
        },
        "invoiceTitle": {
          "type": "keyword"
        },
        "order": {
          "type": "text",
          "fields": {
            "keyword": {
              "ignore_above": 256,
              "type": "keyword"
            }
          }
        },
        "pauseType": {
          "index": false,
          "type": "integer"
        },
        "sellerOrderSum": {
          "type": "long"
        },
        "shouldPay": {
          "type": "long"
        },
        "supposePay": {
          "type": "long"
        },
        "consAddress": {
          "index": false,
          "type": "keyword"
        },
        "idCompanyBranch": {
          "type": "short"
        },
        "version": {
          "type": "text",
          "fields": {
            "keyword": {
              "ignore_above": 256,
              "type": "keyword"
            }
          }
        },
        "idShipmenttype": {
          "type": "short"
        },
        "popSendPayDyTik": {
          "type": "keyword"
        },
        "areaNum": {
          "index": false,
          "type": "integer"
        },
        "paymentForGoodsDyL": {
          "index": false,
          "type": "long"
        },
        "detail": {
          "type": "text",
          "fields": {
            "keyword": {
              "ignore_above": 256,
              "type": "keyword"
            }
          }
        },
        "orderPopStatus": {
          "type": "short"
        },
        "status": {
          "type": "long"
        },
        "city": {
          "type": "integer"
        },
        "freight": {
          "index": false,
          "type": "long"
        },
        "sendpayMap": {
          "index": false,
          "type": "keyword"
        },
        "invoiceState": {
          "type": "integer"
        },
        "orderSignDesc": {
          "type": "text"
        },
        "consName": {
          "type": "keyword"
        },
        "samClubIdDyL": {
          "type": "integer"
        },
        "paymentType": {
          "type": "short"
        },
        "erpOrderStatus": {
          "type": "short"
        },
        "invoiceContentId": {
          "type": "integer"
        },
        "consIndex": {
          "type": "keyword"
        },
        "vatInfo": {
          "index": false,
          "type": "keyword"
        },
        "orderCompletetime": {
          "type": "date"
        },
        "centerId": {
          "type": "long"
        },
        "consPhone": {
          "type": "keyword"
        },
        "orderExt": {
          "index": false,
          "type": "keyword"
        },
        "customerIp": {
          "type": "keyword"
        },
        "pauseDispatch": {
          "type": "integer"
        },
        "storeId": {
          "type": "integer"
        },
        "parentId": {
          "type": "long"
        },
        "virtulMobile": {
          "index": false,
          "type": "keyword"
        },
        "cky2": {
          "type": "short"
        },
        "esModified": {
          "type": "date"
        },
        "deligoodType": {
          "type": "short"
        },
        "returnOrder": {
          "type": "integer"
        },
        "installIdDyS": {
          "type": "keyword"
        },
        "realPin": {
          "type": "keyword"
        },
        "buid": {
          "type": "long"
        },
        "orderSign": {
          "index": false,
          "type": "keyword"
        },
        "pauseBizStatus3cDyI": {
          "type": "integer"
        },
        "salesPinDyS": {
          "type": "keyword"
        },
        "outboundDate": {
          "type": "date"
        },
        "venderIdOrderPopStatus": {
          "type": "keyword"
        },
        "idSopShipmentTypeDyI": {
          "type": "integer"
        },
        "orderSum": {
          "type": "long"
        },
        "tuiHuoWuYouDyL": {
          "type": "integer"
        },
        "venderIdYn": {
          "type": "keyword"
        },
        "pin": {
          "type": "keyword"
        },
        "invoiceType": {
          "type": "keyword"
        },
        "modified": {
          "type": "date"
        },
        "consMobilePhone": {
          "type": "keyword"
        },
        "balanceUsed": {
          "type": "long"
        },
        "town": {
          "type": "integer"
        },
        "created": {
          "type": "date"
        },
        "consEmail": {
          "type": "keyword"
        },
        "venderIdErpOrderStatus": {
          "type": "keyword"
        },
        "menDianIdDyL": {
          "type": "integer"
        },
        "mdbstoreIdDyL": {
          "type": "long"
        },
        "serviceIdDyL": {
          "type": "integer"
        },
        "skus": {
          "type": "nested",
          "properties": {
            "wareId": {
              "index": false,
              "type": "long"
            },
            "givePoint": {
              "type": "long"
            },
            "num": {
              "index": false,
              "type": "integer"
            },
            "itemExt": {
              "index": false,
              "type": "keyword"
            },
            "newStore": {
              "index": false,
              "type": "keyword"
            },
            "purchaseOrderNo": {
              "type": "keyword"
            },
            "skuName": {
              "analyzer": "ik_max_word",
              "type": "text"
            },
            "itemNum": {
              "type": "keyword"
            },
            "refundOrderNo": {
              "type": "keyword"
            },
            "jdPrice": {
              "type": "long"
            },
            "outerId": {
              "index": false,
              "type": "keyword"
            },
            "invoiceContentId": {
              "index": false,
              "type": "long"
            },
            "serviceId": {
              "index": false,
              "type": "long"
            },
            "class": {
              "type": "text",
              "fields": {
                "keyword": {
                  "ignore_above": 256,
                  "type": "keyword"
                }
              }
            },
            "skuId": {
              "type": "long"
            }
          }
        },
        "supplierId": {
          "type": "long"
        },
        "paysumType": {
          "type": "short"
        },
        "venderId": {
          "type": "long"
        },
        "venderIdOrderType": {
          "type": "keyword"
        },
        "remark": {
          "index": false,
          "type": "keyword"
        },
        "logiCopr": {
          "analyzer": "order_analyzer",
          "type": "text"
        },
        "marketId": {
          "type": "long"
        },
        "logiNo": {
          "analyzer": "order_analyzer",
          "type": "text"
        },
        "orderMarkUp": {
          "index": false,
          "type": "keyword"
        },
        "orderCreateDate": {
          "type": "date"
        },
        "total": {
          "type": "long"
        },
        "jiFen": {
          "index": false,
          "type": "integer"
        },
        "serviceFeeDyL": {
          "type": "long"
        },
        "thirdPauseReason": {
          "type": "keyword"
        },
        "class": {
          "type": "text",
          "fields": {
            "keyword": {
              "ignore_above": 256,
              "type": "keyword"
            }
          }
        },
        "pauseBizStatusYyDyI": {
          "type": "integer"
        },
        "ziti": {
          "index": false,
          "type": "short"
        },
        "dbDtDyTdt": {
          "type": "date"
        },
        "pauseStartTime": {
          "type": "date"
        },
        "couponDetails": {
          "type": "nested",
          "properties": {
            "useDate": {
              "index": false,
              "type": "date"
            },
            "rfCode": {
              "index": false,
              "type": "keyword"
            },
            "yn": {
              "index": false,
              "type": "integer"
            },
            "couponType": {
              "index": false,
              "type": "integer"
            },
            "operateUser": {
              "index": false,
              "type": "keyword"
            },
            "couponPrice": {
              "index": false,
              "type": "integer"
            }
          }
        },
        "scDtDyTdt": {
          "type": "date"
        },
        "erpUpdateTime": {
          "index": false,
          "type": "date"
        },
        "esCreated": {
          "type": "date"
        },
        "venderStoreId": {
          "type": "keyword"
        },
        "consPostcode": {
          "index": false,
          "type": "keyword"
        },
        "venderIdRefundFlag": {
          "type": "keyword"
        },
        "carrierId": {
          "type": "keyword"
        },
        "printx": {
          "index": false,
          "type": "short"
        },
        "codDtDyTdt": {
          "type": "date"
        }
      }
    }
  }
}
```
