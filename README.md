# react-native-scatter

最近eos开源联盟有小伙伴问scatter的兼容问题，我在这提供一种简单的思路，此方法同样适用与android/ios原生开发

# 实现一个web用于加载Dapp

```

import RenderScatter from './RenderScatter';  //导入注入的js

export default class Web extends Component {

  //web message
  onMessage = (e) =>{
  
    let result = JSON.parse(e.nativeEvent.data);
    
    //查询余额方法
    if(result.scatter==="getCurrencyBalance"){
      //执行查询余额方法
      this.props.dispatch({type:'wallet/balanceScatter',payload:{...result.params},callback:(res)=>{
        //将结果post出去
        this.refs.refWebview.postMessage(JSON.stringify({...result,data:res}));
      }});
      
    //查询账号信息
    }else if(result.scatter==="getAccount"){
      this.props.dispatch({type:'wallet/getAccount',payload:{...result.params},callback:(res)=>{
        if(res.code==0){
          this.refs.refWebview.postMessage(JSON.stringify({...result,data:res.data}));
        }else{
          Toast.show(res.msg);
        }
      }});
      
    //交易
    }else if(result.scatter==="transaction"){
    
      //弹出交易确认框
      DappTx.show(result.params.actions,()=>{
        account = WalletUtils.selectAccount();
        //弹出输入密码
        Auth.show(account,(pk)=>{
          Loading.show("提交中...");
          //执行交易
          Eos.transaction(pk,result.params.actions,(r)=>{
            Loading.dismis();
            if(r.isSuccess){
              this.refs.refWebview.postMessage(JSON.stringify({...result,data:r.data}));
            }else{
              Toast.show(r.msg);
            }
          })
        },()=>{
          
        });
      });
    }
  }
  
  render(){
    return (
      <View>
        <WebView
          ref="refWebview"
          source={{uri:this.props.navigation.state.params.url}}
          domStorageEnabled={true}
          javaScriptEnabled={true}
          scalesPageToFit={false}
          injectedJavaScript = {RenderScatter(this.props)}  //注入js
          onMessage={(e)=>{this.onMessage(e)}}  //web message
        />
      </View>
    )
  }
}

```

# 注入的JS，RenderScatter

```
export default function RenderScatter(props) {
  let account = WalletUtils.selectAccount(); //获得当前选择的EOS账号
  if(account){
    return `
      iden = {
          name:"${account.name}",
          publicKey:"${account.publicKey}",
          accounts:[{
              name:"${account.name}",
              blockchain:"eos",
              authority:"${account.perm_name}"
          }]
      };
      window.scatter={
          identity:iden,
          getIdentity:function(id){
              return new Promise((resolve, reject) => {
                  resolve(iden);
              })
          },
          eos:(e,t,r,n) =>{
              return {
                  //查询余额
                  getCurrencyBalance:function(contract,name,coin){
                      return new Promise((resolve, reject) => {
                          var key = new Date().getTime();
                          //给WebView发送一个消息，并且吧需要的参数传过去
                          window.postMessage(JSON.stringify({key,scatter:"getCurrencyBalance",params:{contract,name,coin}}));
                          //接收web消息，并将执行结果返回给Dapp
                          document.addEventListener("message",function(msg){
                              document.removeEventListener("message",this);
                              var obj = eval("(" + msg.data + ")");
                              if(obj.scatter==="getCurrencyBalance" && obj.key===key){     
                                  resolve(obj.data);
                              }
                          });
                      })
                  },
                  //查询账号
                  getAccount:function(account){
                      return new Promise((resolve, reject) => {
                          var key = new Date().getTime();
                          //给WebView发送一个消息，并且吧需要的参数传过去
                          window.postMessage(JSON.stringify({key,scatter:"getAccount",params:{account}}));
                          //接收web消息，并将执行结果返回给Dapp
                          document.addEventListener("message",function(msg){
                              document.removeEventListener("message",this);
                              var obj = eval("(" + msg.data + ")");
                              if(obj.scatter==="getAccount" && obj.key===key){     
                                  resolve(obj.data);
                              }
                          });
                      })
                  },
                  //交易
                  transaction:function(actions){
                      return new Promise((resolve, reject) => {
                          var key = new Date().getTime();
                          //给WebView发送一个消息，并且吧需要的参数传过去
                          window.postMessage(JSON.stringify({key,scatter:"transaction",params:{...actions}}));
                          //接收web消息，并将执行结果返回给Dapp
                          document.addEventListener("message",function(msg){
                              document.removeEventListener("message",this);
                              var obj = eval("(" + msg.data + ")");
                              if(obj.scatter==="transaction" && obj.key===key){ 
                                  resolve(obj.data);
                              }
                          });
                      })
                  }
                  //其他更多scatter的eos接口可以在这里实现
              }
          }
      };
      
      //scatter注入成功，发送scatterLoaded Event
      setTimeout(function(){
          var event = document.createEvent('HTMLEvents');
          event.initEvent("scatterLoaded", true, true);
          event.eventType = 'scatterLoaded';
          document.dispatchEvent(event);
      },1000);
  `
  }else{
    return `
    
    `
  }
}

```

# Dapp端使用

```
document.addEventListener('scatterLoaded', scatterExtension => {

  let id = window.scatter.identity;
                
  window.scatter.getIdentity({}).then(identity => {

  });

  const eos = window.scatter.eos("","","",'https');

  eos.transaction([{}]).then(r=>{

  });

  eos.getAccount("eosio").then(r=>{

  });

  eos.getCurrencyBalance('eosio.token',"eosio", 'EOS').then(res =>{

  });

});
```

# join eos open source 

wechat hl_294944589


