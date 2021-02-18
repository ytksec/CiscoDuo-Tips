# CiscoDuo-Tips

Cisco Duo Securityの機能、構成、主な技術解説をまとめていきます。  

## Cisco Duo公式サイト

https://www.cisco.com/c/ja_jp/products/security/adaptive-multi-factor-authentication.html  

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

### オンプレADを認証ソースとする場合

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

## Cisco Umbrellaにてプロキシ認証を実施する場合の構成(SP-iniciate)

[Cisco Japan Blog](https://gblogs.cisco.com/jp/2020/04/secure-remote-work-with-policy-control-with-umbrella-swg-proxy-authentication-and-multi-factor-authentication-with-duo-security/)

## 一般的な２要素認証について

２要素認証といっても、認証の要素は様々あります。一般的に、認証要素としては以下があります。

+ 知識・・・パスワードなど個々のユーザーだけが知っている情報
+ 所有・・・ユーザのみが所有するもの、情報。ハードウェアトークン、モバイルアプリ(Duo Mobileなど)、電話番号(SMS)
+ 固有・・・指紋などユーザのみが持つ要素
+ ロケーション・・・ユーザがログインする場所(グローバルアドレス)
+ 時間・・・ユーザがログインする時間

主に知識、所有、固有情報の中から２要素認証を実施して、ロケーションや時間はポリシーの中で利用されている印象です。

## Duoの２要素認証

2021年1月時点で、以下の2要素認証方法に対応しており、長所、短所を記載していきます。

+ SMS
+ TOTP
+ プッシュベース
+ U2Fトークン
+ WebAuthn
+ 電話コールバック

参考：https://duo.com/product/multi-factor-authentication-mfa/two-factor-authentication-2fa

### SMS
SMSにてワンタイムパスワードが送信され、ユーザーは認証先のWebサイトまたはアプリケーションにコードを入力することで認証が完了します。

+ 長所
シンプルさ、スピード、古くから使われているため受け入れやすい

+ 短所
ユーザは電話番号を保持し、またそれを認証プロバイダに開示させる必要がある。また、SMSを受信できるデバイスを保有している必要があり、ネットワーク(電話回線)へも接続できる環境でなければならない。

### TOTP
時間ベースのワンタイムパスワード。ユーザ固有のQRコードを読み取ると時間ベースのパスコードが発行されるため、コードを入力。

+ 長所・・・
QRコードがあれば、読み取るデバイスに制限がない。また、アプリがあればデバイスはネットワークに接続している必要がない。

+ 短所・・・
ユーザがデバイスやQRコードを忘れると認証が出来ない。

### プッシュベース
Duo Mobileアプリへユーザがログインしようとしている情報（場所、時刻、IPアドレス）をプッシュ送信し、ユーザはプッシュ情報が正しい場合はアクセスを許可。

+ 長所・・・
ユーザは認証のために新たに情報を入力する必要がなく、アプリの通知の情報を確認して許可するのみ。シンプルであり、拡張性も容易。
また、SMSなどは誰でも送付することが出来るためフィッシング詐欺の対象にもなり得るが、この方式の場合はアプリのプッシュ通知のため安全。

+ 短所・・・
アプリをインストールしているモバイルデバイスがネットワークに接続している必要がある。また、ユーザがアプリのプッシュ通知の情報が正しいか確認せずに承認してしまう場合もある。

### U2Fトークン
USBトークンを使用し、ユーザは表示されたPINコードを入力する方式。

+ 長所・・・
ユーザは特別なセットアップを必要とせず、複数のデバイスで利用出来る。また、フィッシング対応にもなる。

+ 短所・・・
物理的なトークンが必要であり、紛失の可能性もある。

### WebAuthn(Web Authentication API)
FIDO（Fast IDentity Online）AllianceとW3Cによって作成されたWeb Authentication APIにより、ユーザはFIDO2準拠のデバイス(Yubikeyなど)で認証。

+ 長所・・・
一番安全な認証方式。生体情報など固有の資格情報を必要とするため、その情報が盗まれる心配が無い。iOSのFace ID/Touch IDにも対応。

+ 短所・・・
WebAuthnは特定のデバイスに設定が紐つけられており、デバイス紛失の際にはアカウント回復が困難。

### 電話コールバック
ユーザのログイン試行があると電話に着信があり、承認、拒否を電話のボタンを押して認証。

+ 長所・・・
ユーザがスマートデバイスを所持しておらず、固定電話のみという場合には有効。

+ 短所・・・
電話の着信に出る、音声を聞くなど時間を要す。電話回線が必要。
