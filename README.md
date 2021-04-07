# CiscoDuo-Tips

Cisco Duo Securityの機能、構成、主な技術解説をまとめていきます。  
あくまで個人的な経験を元に作成している内容となりますので、正式な見解等はメーカードキュメントをご参照ください。

## Cisco Duo公式サイト

https://www.cisco.com/c/ja_jp/products/security/adaptive-multi-factor-authentication.html  

## Duo マニュアル

+ [ユーザーマニュアル](https://guide.duo.com/)・・・ユーザ視点で、インストール方法など詳しく記載されているのでオススメ
+ [設定ドキュメント](https://duo.com/docs/)・・・管理者、エンジニア視点のドキュメント。目当てのページに行きにくいので、ページ下にピックアップしてリンク記載。  
+ [Knowledge Base,Help](https://help.duo.com/s/?language=en_US)
  

## 認証アプリ(Duo Mobile)対応端末

| OS | 対応可否 |
| ------------- | ------------- |
| Windows  | ○  |
| Mac  | ○  |
| iOS  | ○  |
| Android  | ○  |
| Windows Phone  | ○  |
| Blackberry  | ○  |
| Apple Watch  | ○  |

最新情報はこちらを参照。https://guide.duo.com/

## 主な機能

+ ２要素認証  
+ デバイスの可視化  
+ 適応型認証  
+ シングルサインオン（SSO）  
+ アプリケーションの設置場所を選ばない柔軟な保護  
+ セキュアリモートアクセス  

## Duoが提供する認証の流れ

+ ユーザー認証（多要素認証含む）を実施  
+ デバイス認証（適切なデバイスか、健全性判断）を実施  
+ 適応型ポリシー（コンテキストベース）を適用  

## 構成

大前提として、Duoはユーザのパスワード保存は行わず、常に認証ソースへ認証の問い合わせを行います。そのため、どこを認証のソース(idP)とするかは重要な設計要素です。
以下に主要な構成例を紹介していきます。  

### オンプレADを認証ソースとする場合(ディレクトリ、ユーザ同期も可)

オンプレミス(IaaS)設置のADからID情報を連携させる際は、以下のような構成を取ります。
<img width="480" alt="スクリーンショット 2021-01-23 10 26 43" src="https://user-images.githubusercontent.com/76857288/107777296-f6957500-6d85-11eb-9473-aeb5caafa732.png">  
引用：https://duo.com/docs/adsync  
オンプレミス環境にDuo Authentication Proxy(DAP)というソフトウェアをインストールし、DuoとADの連携の役割をさせます。  
DAPはアウトバウンドのみの通信となり、DMZに設置する必要はありません。  

※Duo Authentication ProxyのインストールOS要件  

+ Windows Server 2012 or later (Server 2016 or 2019 recommended)  
+ CentOS 7 or later  
+ Red Hat Enterprise Linux 7 or later  
+ Ubuntu 16.04 or later  
+ Debian 7 or later.  

※必要サーバスペック  

at least 1 CPU, 200 MB disk space, and 4 GB RAM (although 1 GB RAM is usually sufficient)  

参考元：https://duo.com/docs/authproxy-reference  

この構成が出来上がると、以下のような構成でユーザー認証、SSO先へのログインができるようになります。  

<img width="777" alt="スクリーンショット 2021-01-23 10 37 16" src="https://user-images.githubusercontent.com/76857288/107777807-ad91f080-6d86-11eb-8f2e-19182403f4c4.png">  

引用元：https://duo.com/docs/sso  
ユーザはSSO（SAML連携）先として登録しているSaaSなどのサイト(SP)へログインを試みると、Duoのログイン画面に遷移し、裏でADとの認証、MFAを実施した上でサイトへ安全にログインすることが出来ます。

以前はDuo Access Gateway(DAG)が必要でしたが、2020年末よりCloud SSO機能がリリースされたため、オンプレADとの連携の際にDMZにDAGを設置する必要がなくなりました。  
これにより、DAGを公開するというリスクが減り、よりDuoを導入しやすくなったのではないかと思います。  

### Duo Authentication Proxy(DAP)について  
基本的に冗長化が推奨　[Duo knowledge](https://help.duo.com/s/article/authentication-proxy-availability?language=en_US)  
DAPのAD参加については必須ではない　https://help.duo.com/s/article/3329?language=en_US  

### 各SAML Identity Providerを認証ソースとする場合

![構成イメージ](https://github.com/ytksec/CiscoDuo-Tips/blob/main/img/Duo-SAMLidP.jpg)  
引用：https://duo.com/docs/sso  

Duoは各SAML Identity Providerを認証ソースとしてシングルサインオンをすることが可能。  
Okta、Azure AD、ADFS、Ping Identity、One Login、他Identity Providerと連携を行うことが出来る。

イメージとして、DuoとBoxでSAML連携設定しておき、SSOのプライマリ認証をOktaとしていた場合、
BoxからSSO認証しようとすると、Oktaの画面へ遷移しログインを実施する形になる。  
[設定例はこちら](https://duo.com/docs/sso)

所感では、Oktaや他idPを主で管理している場合は、各SaaSへのSAML設定も一緒に行っている場合が多いのではないかと思われる。  
その場合、Duoの認証ソースとしてidPを利用するのではなく、DuoをMFAツールとして利用するという選択肢もある。  
ただし、Duoのアプリケーション保護でグループやデバイスごとに細かく制御を分けたい、Duoのデバイストラスト等の機能を利用したい等であれば、この構成のようにDuoをから各アプリケーションをSAML連携させる形が良いと思われる。

### Azure ADとディレクトリ同期をする場合

![構成イメージ](https://github.com/ytksec/CiscoDuo-Tips/blob/main/img/Duo-AzureAD.jpg)  
引用：https://duo.com/docs/azuresync

Azure ADとディレクトリ、ユーザ同期をすることが可能であり、オンプレADと同期をするよりも可用性もあり、推奨。  
[設定方法はこちらを参照](https://duo.com/docs/azuresync)

## 2要素認証について

こちらにまとめてみました。  
https://github.com/ytksec/CiscoDuo-Tips/blob/main/Duo-2factor-auth.md

## Cisco Anyconnectを用いてDuoとASA(Firepower)のSSOを実現するための構成例

+ [構成例はこちら](https://duo.com/docs/cisco)
+ [ASAの設定例はこちら](https://duo.com/docs/sso-ciscoasa)

## Cisco UmbrellaをSP、DuoをidPとした場合のSAML構成例(SP-iniciate)

[Cisco Japan Blog](https://gblogs.cisco.com/jp/2020/04/secure-remote-work-with-policy-control-with-umbrella-swg-proxy-authentication-and-multi-factor-authentication-with-duo-security/)

## その他設定関連のDuoドキュメント
+ [Trust Monitor](https://duo.com/docs/trust-monitor)
+ [Duo Sigle Sign On](https://duo.com/docs/sso)
