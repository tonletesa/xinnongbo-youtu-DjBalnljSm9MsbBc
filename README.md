在触达用户的多种途径中，推送通知消息凭借其高效性和便捷性，成为一种高性价比的营销手段。然而由于各应用推送频率过高，导致重要通知消息常被淹没在海量信息中，难以及时触达用户。比如商家的新订单提醒或者是收款到账通知等重要提醒，往往会因为消息过多而被用户忽视。

为解决这一问题，确保重要消息能够及时精准触达用户，HarmonyOS SDK[推送服务](https://github.com "推送服务")（Push Kit）提供了推送通知扩展消息功能，该功能支持通过语音播报的方式，让用户能够迅速感知到重要消息，解锁全新的通知体验。

![图片1](https://img2024.cnblogs.com/blog/2396482/202509/2396482-20250917112904509-1709446604.png)

在发送通知扩展消息的过程中，存在应用进程在前台和不在前台两种情况。当用户终端收到应用发送的通知扩展消息时：

若应用进程不在前台，Push Kit会将消息内容传递给通知扩展进程，开发者可以在该进程中自行完成语音播报业务处理后，返回自定义消息内容，Push Kit将弹出通知提醒。同时，返回消息内容需要在10秒内完成，否则Push Kit将默认展示原有的消息内容。

若应用进程在前台，则不弹出通知提醒，开发者可以在应用进程中获取通知扩展消息内容并自行完成语音播报业务处理。

下面，我们来看一下推送通知扩展消息的具体开发步骤。

**开发步骤**

1. 首先，在发送任何类型的消息之前，都需要先获取Push Token。

```
import { pushService } from '@kit.PushKit';
    import { hilog } from '@kit.PerformanceAnalysisKit';
    import { BusinessError } from '@kit.BasicServicesKit';
    import { UIAbility, AbilityConstant, Want } from '@kit.AbilityKit';

    export default class EntryAbility extends UIAbility {
      // 入参 want 与 launchParam 并未使用，为初始化项目时自带参数
      async onCreate(want: Want, launchParam: AbilityConstant.LaunchParam): Promise {
        // 获取Push Token
        try {
          const pushToken: string = await pushService.getToken();
          hilog.info(0x0000, 'testTag', 'Succeeded in getting push token');
        } catch (err) {
          let e: BusinessError = err as BusinessError;
          hilog.error(0x0000, 'testTag', 'Failed to get push token: %{public}d %{public}s', e.code, e.message);
        }
        // 上报Push Token并上报到您的服务端
      }
    }
```

2. 为确保应用可正常收到消息，建议应用发送通知前调用requestEnableNotification()方法弹出提醒，告知用户需要允许接收通知消息。

```
 import { notificationManager } from '@kit.NotificationKit';
    import { BusinessError } from '@kit.BasicServicesKit';
    import { hilog } from '@kit.PerformanceAnalysisKit';
    import { common } from '@kit.AbilityKit';

    const TAG: string = '[PublishOperation]';
    const DOMAIN_NUMBER: number = 0xFF00;
    let context = this.getUIContext().getHostContext() as common.UIAbilityContext;
    notificationManager.isNotificationEnabled().then((data: boolean) => {
      hilog.info(DOMAIN_NUMBER, TAG, "isNotificationEnabled success, data: " + JSON.stringify(data));
      if(!data){
        notificationManager.requestEnableNotification(context).then(() => {
          hilog.info(DOMAIN_NUMBER, TAG, `[ANS] requestEnableNotification success`);
        }).catch((err : BusinessError) => {
          if(1600004 == err.code){
            hilog.error(DOMAIN_NUMBER, TAG, `[ANS] requestEnableNotification refused, code is ${err.code}, message is ${err.message}`);
          } else {
            hilog.error(DOMAIN_NUMBER, TAG, `[ANS] requestEnableNotification failed, code is ${err.code}, message is ${err.message}`);
          }
        });
      }
    }).catch((err : BusinessError) => {
        hilog.error(DOMAIN_NUMBER, TAG, `isNotificationEnabled fail, code is ${err.code}, message is ${err.message}`);
    });
```

3. 然后在工程内创建一个ExtensionAbility类型的组件并且继承RemoteNotificationExtensionAbility，完成onReceiveMessage()方法的覆写。

```
    import { pushCommon, RemoteNotificationExtensionAbility } from '@kit.PushKit';
    import { image } from '@kit.ImageKit';
    import { hilog } from '@kit.PerformanceAnalysisKit';
    import { resourceManager } from '@kit.LocalizationKit';
    import { common } from '@kit.AbilityKit';

    export default class RemoteNotificationExtAbility extends RemoteNotificationExtensionAbility {
      async onReceiveMessage(remoteNotificationInfo: pushCommon.RemoteNotificationInfo): Promise {
        hilog.info(0x0000, 'testTag', 'RemoteNotificationExtAbility onReceiveMessage, remoteNotificationInfo');

        // Read the pixel map object
        const resourceMgr: resourceManager.ResourceManager = (this.context as common.UIExtensionContext).resourceManager;
        const fileData: Uint8Array = await resourceMgr.getMediaContent($r('app.media.icon'));
        const buffer = fileData.buffer;
        const imageSource: image.ImageSource = image.createImageSource(buffer as ArrayBuffer);
        const pixelMap: image.PixelMap = await imageSource.createPixelMap();
        if (pixelMap) {
          pixelMap.getImageInfo((err, imageInfo) => {
            if (imageInfo) {
              hilog.info(0x0000, 'testTag', `imageInfo ${imageInfo.size.width} * ${imageInfo.size.height}`);
            }
          });
        }

        // Return the replaced message content.
        return {
          title: 'Default replace title.',
          text: 'Default replace text.',
          badgeNumber: 1,
          setBadgeNumber: 2,
          overlayIcon: pixelMap,
          wantAgent: {
            abilityName: 'DemoAbility',
            parameters: {
              key: 'Default value'
            }
          }
        }
      }

      onDestroy(): void {
        hilog.info(0x0000, 'testTag', 'RemoteNotificationExtAbility onDestroy.');
      }
    }
```

4. 接着在项目工程的src/main/module.json5文件的extensionAbilities模块中配置RemoteNotificationExtAbility的type和actions信息。需要注意的是，定义该type和actions的ExtensionAbility有且只能有一个，若同时添加uris参数，则uris内容需为空。

```
    "extensionAbilities": [
      {
        "name": "RemoteNotificationExtAbility",
        "type": "remoteNotification",
        "srcEntry": "./ets/entryability/RemoteNotificationExtAbility.ets",
        "description": "RemoteNotificationExtAbility test",
        "exported": false,
        "skills": [
          {
            "actions": ["action.hms.push.extension.remotenotification"]
          }
        ]
      }
    ]
```

5. 最后，应用服务端调用REST API推送扩展消息。在示例代码中，push-type为2，表示消息类型为通知扩展消息，extraData后的字符串为通知扩展场景可携带的额外数据。

```
    // Request URL 
    POST https://push-api.cloud.huawei.com/v3/[projectId]/messages:send
     
    // Request Header 
    Content-Type: application/json
    Authorization: Bearer eyJr*****OiIx---****.eyJh*****iJodHR--***.QRod*****4Gp---**** 
    push-type: 2

    // Request Body
    { 
      "payload": { 
        "extraData": "通知扩展场景携带的额外数据", 
        "notification": { 
          "category": "EXPRESS",  
          "sound": "alert.mp3",
          "soundDuration": 40,
          "title": "通知标题", 
          "body": "通知内容", 
          "clickAction": { 
            "actionType": 0 
          },
          "notifyId": 12345,
          "image": "https://***.png"
        }
      }, 
      "target": { 
        "token": ["IQAAAACy0tEjCgBijrEB3************8o0m5EdTXbdlhiIiX_vNGQ5Ic5rXWmw"] 
      }, 
      "pushOptions": { 
        "testMessage": true,
        "ttl": 86400
      } 
    }
```

如果消息体中携带extraData字段，那么在设备端侧收到消息后需要传递数据至客户端应用，我们需要通过receiveMessage()方法实时获取通知扩展消息数据extraData。

```
   import { UIAbility } from '@kit.AbilityKit';
    import { pushService } from '@kit.PushKit';
    import { BusinessError } from '@kit.BasicServicesKit';
    import { hilog } from '@kit.PerformanceAnalysisKit';

    export interface ExtraData {
      title: string;
      text: string;
    }

    /**
     * 此处以PushMessageAbility为例，接收通知扩展消息内容
     */
    export default class PushMessageAbility extends UIAbility {
      onCreate(): void {
        try {
          // receiveMessage中的参数固定为IM
          pushService.receiveMessage('IM', this, (payload) => {
            // process message，并建议对Callback进行try-catch
            try {
              // get extraData passed by REST API
              const extraData: ExtraData = JSON.parse(JSON.parse(payload.data).data);
              hilog.info(0x0000, 'testTag', 'Succeeded in getting extraData,data=%{public}s', JSON.stringify(extraData));
            } catch (e) {
              let errRes: BusinessError = e as BusinessError;
              hilog.error(0x0000, 'testTag', 'Failed to process data: %{public}d %{public}s', errRes.code, errRes.message);
            }
          });
        } catch (err) {
          let e: BusinessError = err as BusinessError;
          hilog.error(0x0000, 'testTag', 'Failed to get message: %{public}d %{public}s', e.code, e.message);
        }
      }
    }
```

6. 在发送通知扩展消息后，若应用进程不在前台，Push Kit会将通知消息内容传递给通知扩展进程，您在该进程中自行完成语音播报业务处理，并返回特定的消息内容（例如title、body等）后，Push Kit将弹出通知提醒。若您的应用进程在前台，则不弹出通知提醒。

在项目模块的src/main/module.json5文件的abilities模块中配置skills标签的actions属性内容为 action.ohos.push.listener。

```
  {
      "name": "PushMessageAbility",
      "srcEntry": "./ets/abilities/PushMessageAbility.ets",
      "launchType": "singleton",
      "startWindowIcon": "$media:startIcon",
      "startWindowBackground": "$color:start_window_background",
      "exported": false,
      "skills": [
        {
          "actions": [
            "action.ohos.push.listener"
          ]
        }
      ]
    }
```

**了解更多详情>>**

访问[推送服务联盟官网](https://github.com "推送服务联盟官网"):[虎跃加速](https://huyuejiasu.com)

获取[推送通知扩展消息开发指导文档](https://github.com "推送通知扩展消息开发指导文档")
