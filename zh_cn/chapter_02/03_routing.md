# 路由功能

V2Ray 内建了一个简单的路由功能，可以将传入数据按需求由不同的传出连接发出，以达到按需翻墙的目的。这一功能的常见用法是建立一个统一的中转服务器（比如在路由器或者一台国内的 VPS 上），所有的客户端都将数据发往这台服务器，由服务器来选择是否转发至国外的 VPS。这样做的好处是减化了客户端的配置和维护成本，当路由有变化时，不必修改每一个客户端，只需修改中转服务器的配置即可。

目前路由配置只支持 strategy = "rules"，配置格式如下：

```javascript
{
  "domainStrategy": "AsIs",
  "rules": [
    {
      "type": "field",
      "domain": [
        "baidu.com",
        "qq.com"
      ],
      "outboundTag": "direct"
    },
    {
      "type": "field",
      "ip": "0.0.0.0/8",
      "outboundTag": "direct"
    },
    {
      "type": "field",
      "network": "udp",
      "outboundTag": "blocked"
    }
  ]
}
```

其中：
* domainStrategy (V2Ray 1.13+): 域名解析策略，可选的值有：
  * "AsIs": 只使用域名进行路由选择。默认值。
  * "IPIfNonMatch": 当域名没有匹配任何规则时，将域名解析成 IP（A 记录）再次进行匹配；
    * 当一个域名有多个 A 记录时，会尝试匹配所有的 A 记录，直到其中一个与某个规则匹配为止；
    * 解析后的 IP 仅在路由选择时起作用，转发的数据包中依然使用原始域名；
* rules: 对应一个数组，数组中每个一个元素是一个规则。对于每一个 TCP/UDP 连接，路由将根据这些规则依次进行判断，当一个规则生效时，即将这个连接按此规则的设置进行转发。

每一个规则都有两个必须的属性： type 和 outboundTag。type 表示此规则的类型，目前支持的类型有：field、chinaip 和 chinasites；outboundTag 对应一个[额外传出连接配置](02_protocols.md)的标识。

三种类型的详细格式如下：

### field
```javascript
{
  "type": "field",
  "domain": [
    "baidu.com",
    "qq.com"
  ],
  "ip": [
    "0.0.0.0/8",
    "10.0.0.0/8",
    "fc00::/7",
    "fe80::/10"
  ],
  "port": "0-100",
  "network": "tcp",
  "outboundTag": "direct"
}
```
其中：
* domain: 一个数组，数组每一项是一个域名的匹配。有两种形式：
  * 纯字符串: 当此字符串匹配目标域名中任意部分，该规则生效。比如"sina.com"可以匹配"sina.com"、"sina.com.cn"和"www.sina.com"，但不匹配"sina.cn"。
  * 正则表达式: 由"regexp:"，余下部分是一个正则表达式。当此正则表达式匹配目标域名时，该规则生效。例如"regexp:\\\\.goo.*\\\\.com$"匹配"www.google.com"、"fonts.googleapis.com"，但不匹配"google.com"。
* ip：一个数组，数组内每一个元素是一个 [CIDR](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing)。当某一元素匹配目标 IP 时，此规则生效。
* port：端口范围，有两种形式：
  * "a-b": a 和 b 均为正整数，且小于 65536。这个范围是一个前后闭合区间，当目标端口落在此范围内时，此规则生效。
  * a: a 为正整数，且小于 65536。当目标端口为 a 时，此规则生效。
* network: 可选的值有"tcp"、"udp"或"tcp,udp"，当连接方式是指定的方式时，此规则生效。

一些特殊情况：
* 当多个属性同时指定时，这些属性需要同时满足，才可以使当前规则生效；
* 当一个网络连接指定了域名，而一个规则中只有 IP 规则，没有域名规则，则这个规则永远不会生效，反之亦然。


### chinaip
此规则配置如下，没有额外属性。
```javascript
{
  "type": "chinaip",
  "outboundTag": "direct"
}
```

chinaip 会匹配所有中国境内的 IP，目前只包含 IPv4 数据。IP 段信息由 [ChinaDNS](https://github.com/shadowsocks/ChinaDNS) 中给出的脚本生成。

### chinasites
此规则配置如下，没有额外属性。
```javascript
{
  "type": "chinasites",
  "outboundTag": "direct"
}
```

chinasites 内置了一些[常见的国内网站域名](https://github.com/v2ray/v2ray-core/blob/master/app/router/rules/chinasites.go)，如果目标域名为这些域名其中之一时，此规则生效。
