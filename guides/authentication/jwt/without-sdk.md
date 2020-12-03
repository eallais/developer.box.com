---
rank: 2
related_endpoints:
  - get_authorize
related_guides:
  - applications/select
  - authentication/select
  - applications/custom-apps/oauth2-setup
required_guides:
  - authentication/select
  - applications/custom-apps/oauth2-setup
related_resources: []
alias_paths:
  - /docs/construct-jwt-claim-manually
  - /guides/authentication/client-credentials
category_id: authentication
subcategory_id: authentication/jwt
is_index: false
id: authentication/jwt/without-sdk
type: guide
total_steps: 4
sibling_id: authentication/jwt
parent_id: authentication/jwt
next_page_id: authentication/jwt/as-user
previous_page_id: authentication/jwt/with-sdk
source_url: >-
  https://github.com/box/developer.box.com/blob/main/content/guides/authentication/jwt/without-sdk.md
fullyTranslated: true
---
# SDKを使用しないJWT

This guide takes you through JWT authentication without using a Box SDK. JWT does not require end-user interaction and is designed to authenticate directly with the Box API.

There are two ways you can verify an application's permissions:

* using a [public and private key pair][keypair]
* using a [client id and client secret][ccg] (Client Credentials Grant)

Using a Client Credentials Grant is the easiest and fastest verification method for prototyping or scripting against your Box enterprise. Regardless of the verification method you select from above, the end result is an Access Token which can be used to make API calls on behalf of the application. 

To learn how to use this token visit our guide on [Making API calls](g://api-calls).

<Message notice>

By default, the Access Token acquired with JWT is tied to the Service Account for the application. Any API call made with this token will come from the application. This account does not have access to any existing files or folders until the application's Service Account is added as a collaborator.

`as-user`ヘッダーを使用するか[ユーザーアクセストークン](g://authentication/jwt/user-access-tokens)をリクエストして、[別のユーザーとして処理を実行](g://authentication/oauth2/as-user)できます。

</Message>

## Using a key pair

Follow the steps below if you would like to verify your application's identity using a public and private key pair.

### 前提条件

* A Custom Application using JWT authentication within the Box [Developer Console][devconsole]
* A private key [configuration file][configfile] named `config.json`
* Ensure your application is authorized within the Box Admin Console

### 1. JSON構成を読み取る

Your `config.json` file contains the application's private key and other details necessary to authenticate. The following is an example of this file.

<Tabs>

<Tab title="config.json">

<!-- markdownlint-disable line-length -->

```json
{
  "boxAppSettings": {
    "clientID": "abc...123",
    "clientSecret": "def...234",
    "appAuth": {
      "publicKeyID": "abcd1234",
      "privateKey": "-----BEGIN ENCRYPTED PRIVATE KEY-----\n....\n-----END ENCRYPTED PRIVATE KEY-----\n",
      "passphrase": "ghi...345"
    }
  },
  "enterpriseID": "1234567"
}
```

<!-- markdownlint-enable line-length -->

</Tab>

</Tabs>

このオブジェクトをアプリケーションで使用するには、ファイルから読み取る必要があります。

<Tabs>

<Tab title=".Net">

```dotnet
using System;
using System.IO;
using Newtonsoft.Json;

class Config
{
    public class BoxAppSettings {
        public class AppAuth {
            public string privateKey { get; set; }
            public string passphrase { get; set; }
            public string publicKeyID { get; set; }
        }
        public string clientID { get; set; }
        public string clientSecret { get; set; }
        public AppAuth appAuth { get; set; }

    }
    public string enterpriseID { get; set; }
    public BoxAppSettings boxAppSettings { get; set; }
}

var reader = new StreamReader("config.json");
var json = reader.ReadToEnd();

var config = JsonConvert.DeserializeObject<Config>(json);
```

</Tab>

<Tab title="Java">

```java
import java.io.FileReader;

import com.google.gson.Gson;
import com.google.gson.GsonBuilder;

class Config {
  class BoxAppSettings {
    class AppAuth {
      String privateKey;
      String passphrase;
      String publicKeyID;
    }

    String clientID;
    String clientSecret;
    AppAuth appAuth;
  }

  BoxAppSettings boxAppSettings;
  String enterpriseID;
}

FileReader reader = new FileReader("config.json");

Gson gson = new GsonBuilder().create();
Config config = (Config) gson.fromJson(reader, Config.class);
```

</Tab>

<Tab title="Python 3">

```python
import json
import os

config = json.load(open('config.json'))
```

</Tab>

<Tab title="Node">

```js
const fs = require("fs");

const config = JSON.parse(fs.readFileSync("config.json"));
```

</Tab>

<Tab title="Ruby">

```ruby
require 'json'

config = JSON.parse(
  File.read('config.json')
)
```

</Tab>

<Tab title="PHP">

```php
$json = file_get_contents('config.json');
$config = json_decode($json);
```

</Tab>

</Tabs>

<Message>

# JSONの解析

プログラミング言語によっては、ファイルからJSONを読み取って解析する方法が複数ある場合があります。エラー処理など、さらに詳細な説明については、使用するプログラミング言語のガイドを参照してください。

</Message>

### 2. 秘密キーを復号化する

JWTアサーションを作成するために、アプリケーションでは構成オブジェクトにある秘密キーが必要になります。この秘密キーは暗号化されており、ロックを解除するにはパスコードが必要です。暗号化されたキーとパスコードは両方とも、構成オブジェクトで指定されています。

<Tabs>

<Tab title=".Net">

```dotnet
using System.Security.Cryptography;
using Org.BouncyCastle.OpenSsl;
using Org.BouncyCastle.Crypto.Parameters;
using Org.BouncyCastle.Math;

// https://www.bouncycastle.org/csharp/index.html
class PasswordFinder : IPasswordFinder
{
  private string password;
  public PasswordFinder(string _password) { password = _password; }
  public char[] GetPassword() { return password.ToCharArray(); }
}

var appAuth = config.boxAppSettings.appAuth;
var stringReader = new StringReader(appAuth.privateKey);
var passwordFinder = new PasswordFinder(appAuth.passphrase);
var pemReader = new PemReader(stringReader, passwordFinder);
var keyParams = (RsaPrivateCrtKeyParameters) pemReader.ReadObject();

public RSA CreateRSAProvider(RSAParameters rp)
{
  var rsaCsp = RSA.Create();
  rsaCsp.ImportParameters(rp);
  return rsaCsp;
}

public RSAParameters ToRSAParameters(RsaPrivateCrtKeyParameters privKey)
{
  RSAParameters rp = new RSAParameters();
  rp.Modulus = privKey.Modulus.ToByteArrayUnsigned();
  rp.Exponent = privKey.PublicExponent.ToByteArrayUnsigned();
  rp.P = privKey.P.ToByteArrayUnsigned();
  rp.Q = privKey.Q.ToByteArrayUnsigned();
  rp.D = ConvertRSAParametersField(privKey.Exponent, rp.Modulus.Length);
  rp.DP = ConvertRSAParametersField(privKey.DP, rp.P.Length);
  rp.DQ = ConvertRSAParametersField(privKey.DQ, rp.Q.Length);
  rp.InverseQ = ConvertRSAParametersField(privKey.QInv, rp.Q.Length);
  return rp;
}

public byte[] ConvertRSAParametersField(BigInteger n, int size)
{
  byte[] bs = n.ToByteArrayUnsigned();
  if (bs.Length == size)
      return bs;
  if (bs.Length > size)
      throw new ArgumentException("Specified size too small", "size");
  byte[] padded = new byte[size];
  Array.Copy(bs, 0, padded, size - bs.Length, bs.Length);
  return padded;
}

var key = CreateRSAProvider(ToRSAParameters(keyParams));
```

</Tab>

<Tab title="Java">

```java
import java.io.StringReader;
import java.security.PrivateKey;
import java.security.Security;

import org.bouncycastle.asn1.pkcs.PrivateKeyInfo;
import org.bouncycastle.jce.provider.BouncyCastleProvider;
import org.bouncycastle.openssl.PEMParser;
import org.bouncycastle.openssl.jcajce.JcaPEMKeyConverter;
import org.bouncycastle.openssl.jcajce.JceOpenSSLPKCS8DecryptorProviderBuilder;
import org.bouncycastle.operator.InputDecryptorProvider;
import org.bouncycastle.pkcs.PKCS8EncryptedPrivateKeyInfo;

// https://www.bouncycastle.org/java.html
Security.addProvider(new BouncyCastleProvider());

PEMParser pemParser = new PEMParser(
  new StringReader(config.boxAppSettings.appAuth.privateKey)
);
Object keyPair = pemParser.readObject();
pemParser.close();

char[] passphrase = config.boxAppSettings.appAuth.passphrase.toCharArray();
JceOpenSSLPKCS8DecryptorProviderBuilder decryptBuilder =
  new JceOpenSSLPKCS8DecryptorProviderBuilder().setProvider("BC");
InputDecryptorProvider decryptProvider
  = decryptBuilder.build(passphrase);
PrivateKeyInfo keyInfo
  = ((PKCS8EncryptedPrivateKeyInfo) keyPair).decryptPrivateKeyInfo(decryptProvider);

PrivateKey key = (new JcaPEMKeyConverter()).getPrivateKey(keyInfo);
```

</Tab>

<Tab title="Python 3">

```python
from cryptography.hazmat.backends import default_backend
from cryptography.hazmat.primitives.serialization import load_pem_private_key

appAuth = config["boxAppSettings"]["appAuth"]
privateKey = appAuth["privateKey"]
passphrase = appAuth["passphrase"]

# https://cryptography.io/en/latest/
key = load_pem_private_key(
  data=privateKey.encode('utf8'),
  password=passphrase.encode('utf8'),
  backend=default_backend(),
)
```

</Tab>

<Tab title="Node">

```js
let key = {
  key: config.boxAppSettings.appAuth.privateKey,
  passphrase: config.boxAppSettings.appAuth.passphrase
};
```

</Tab>

<Tab title="Ruby">

```ruby
require "openssl"

appAuth = config['boxAppSettings']['appAuth']
key = OpenSSL::PKey::RSA.new(
  appAuth['privateKey'],
  appAuth['passphrase']
)
```

</Tab>

<Tab title="PHP">

```php
$private_key = $config->boxAppSettings->appAuth->privateKey;
$passphrase = $config->boxAppSettings->appAuth->passphrase;
$key = openssl_pkey_get_private($private_key, $passphrase);
```

</Tab>

</Tabs>

<Message>

# ファイルから秘密キーを読み込むための代替方法

アプリケーションでは、秘密キーとパスワードの両方をディスクに保存しておきたくない場合があります。代替方法として、パスワードを環境変数として渡し、秘密キーを、秘密キーのロックを解除するためのトークンと分けておくこともできます。

</Message>

### 3. JWTアサーションを作成する

To authenticate to the Box API the application needs to create a signed JWT assertion that can be exchanged for an Access Token.

A JWT assertion is an encrypted JSON object, consisting of a `header`, `claims`, and `signature`. Let's start by creating the `claims`, sometimes also referred to as the `payload`.

<Tabs>

<Tab title=".Net">

```dotnet
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Collections.Generic;

byte[] randomNumber = new byte[64];
RandomNumberGenerator.Create().GetBytes(randomNumber);
var jti = Convert.ToBase64String(randomNumber);

DateTime expirationTime = DateTime.UtcNow.AddSeconds(45);

var claims = new List<Claim>{
  new Claim("sub", config.enterpriseID),
  new Claim("box_sub_type", "enterprise"),
  new Claim("jti", jti),
};
```

</Tab>

<Tab title="Java">

```java
import org.jose4j.jwt.JwtClaims;

String authenticationUrl = "https://api.box.com/oauth2/token";

JwtClaims claims = new JwtClaims();
claims.setIssuer(config.boxAppSettings.clientID);
claims.setAudience(authenticationUrl);
claims.setSubject(config.enterpriseID);
claims.setClaim("box_sub_type", "enterprise");
claims.setGeneratedJwtId(64);
claims.setExpirationTimeMinutesInTheFuture(0.75f);
```

</Tab>

<Tab title="Python 3">

```python
import time
import secrets

authentication_url = 'https://api.box.com/oauth2/token'

claims = {
  'iss': config['boxAppSettings']['clientID'],
  'sub': config['enterpriseID'],
  'box_sub_type': 'enterprise',
  'aud': authentication_url,
  'jti': secrets.token_hex(64),
  'exp': round(time.time()) + 45
}
```

</Tab>

<Tab title="Node">

```js
const crypto = require("crypto");

const authenticationUrl = "https://api.box.com/oauth2/token";

let claims = {
  iss: config.boxAppSettings.clientID,
  sub: config.enterpriseID,
  box_sub_type: "enterprise",
  aud: authenticationUrl,
  jti: crypto.randomBytes(64).toString("hex"),
  exp: Math.floor(Date.now() / 1000) + 45
};
```

</Tab>

<Tab title="Ruby">

```ruby
require 'securerandom'

authentication_url = 'https://api.box.com/oauth2/token'

claims = {
  iss: config['boxAppSettings']['clientID'],
  sub: config['enterpriseID'],
  box_sub_type: 'enterprise',
  aud: authentication_url,
  jti: SecureRandom.hex(64),
  exp: Time.now.to_i + 45
}
```

</Tab>

<Tab title="PHP">

```php
$authenticationUrl = 'https://api.box.com/oauth2/token';

$claims = [
  'iss' => $config->boxAppSettings->clientID,
  'sub' => $config->enterpriseID,
  'box_sub_type' => 'enterprise',
  'aud' => $authenticationUrl,
  'jti' => base64_encode(random_bytes(64)),
  'exp' => time() + 45,
  'kid' => $config->boxAppSettings->appAuth->publicKeyID
];
```

</Tab>

</Tabs>

<!-- markdownlint-disable line-length -->

| パラメータ              | 型       | 説明                                                                                        |
| ------------------ | ------- | ----------------------------------------------------------------------------------------- |
| `iss`(必須)          | String  | BoxアプリケーションのOAuthクライアントID                                                                 |
| `sub`(必須)          | String  | Box Enterprise ID (このアプリがそのアプリケーションのサービスアカウントの代わりになる場合)またはユーザーID (このアプリが別のユーザーの代わりになる場合)。 |
| `box_sub_type`(必須) | String  | `enterprise`または`user` (`sub`クレームでリクエストされているトークンの種類に応じて決定)                                 |
| `aud`(必須)          | String  | 常に`https://api.box.com/oauth2/token`                                                      |
| `jti`(必須)          | String  | このJWTに対してアプリケーションで指定されたUUID (Universally Unique Identifier)。16文字以上128文字以下の一意の文字列です。       |
| `exp`(必須)          | Integer | このJWTが期限切れとなるUnix時間。設定できる最大値は、発行時刻から60秒後です。許容される最大値よりも小さい値を設定することをお勧めします。                 |
| `iat`(省略可)         | Integer | 発行時刻。トークンは、この時刻より前に使用することはできません。                                                          |
| `nbf`(省略可)         | Integer | 開始時刻。トークンの有効期間の開始時刻を指定します。                                                                |

<!-- markdownlint-enable line-length -->

次に、秘密キーを使用してこれらのクレームに署名する必要があります。使用する言語とライブラリに応じて、クレームの署名に使用する暗号化アルゴリズムと公開キーのIDを定義することで、JWTの`header`が構成されます。

<Tabs>

<Tab title=".Net">

```dotnet
using Microsoft.IdentityModel.Tokens;

String authenticationUrl = "https://api.box.com/oauth2/token";

var payload = new JwtPayload(
  config.boxAppSettings.clientID,
  authenticationUrl,
  claims,
  null,
  expirationTime
);

var credentials = new SigningCredentials(
  new RsaSecurityKey(key),
  SecurityAlgorithms.RsaSha512
);
var header = new JwtHeader(signingCredentials: credentials);

var jst = new JwtSecurityToken(header, payload);
var tokenHandler = new JwtSecurityTokenHandler();
string assertion = tokenHandler.WriteToken(jst);
```

</Tab>

<Tab title="Java">

```java
import org.jose4j.jws.AlgorithmIdentifiers;
import org.jose4j.jws.JsonWebSignature;

JsonWebSignature jws = new JsonWebSignature();
jws.setPayload(claims.toJson());
jws.setKey(key);

jws.setAlgorithmHeaderValue(AlgorithmIdentifiers.RSA_USING_SHA512);
jws.setHeader("typ", "JWT");
jws.setHeader("kid", config.boxAppSettings.appAuth.publicKeyID);
String assertion = jws.getCompactSerialization();
```

</Tab>

<Tab title="Python 3">

```python
import jwt

keyId = config['boxAppSettings']['appAuth']['publicKeyID']

assertion = jwt.encode(
  claims,
  key,
  algorithm='RS512',
  headers={
    'kid': keyId
  }
)
```

</Tab>

<Tab title="Node">

```js
const jwt = require('jsonwebtoken')

let keyId = config.boxAppSettings.appAuth.publicKeyID

let headers = {
  'algorithm': 'RS512',
  'keyid': keyId,
}

let assertion = jwt.sign(claims, key, headers)
```

</Tab>

<Tab title="Ruby">

```ruby
require 'jwt'
keyId = appAuth['publicKeyID']
assertion = JWT.encode(claims, key, 'RS512', { kid: keyId })
```

</Tab>

<Tab title="PHP">

```php
use \Firebase\JWT\JWT;
$assertion = JWT::encode($claims, $key, 'RS512');
```

</Tab>

</Tabs>

ヘッダーでは、以下のパラメータがサポートされます。

<!-- markdownlint-disable line-length -->

| パラメータ           | 型      | 説明                                                               |
| --------------- | ------ | ---------------------------------------------------------------- |
| `algorithm`(必須) | String | JWTクレームへの署名に使用する暗号化アルゴリズム。RS256、RS384、RS512のいずれかを指定できます。         |
| `keyid`(必須)     | String | JWTへの署名に使用する公開キーのID。必須ではありませんが、アプリケーションに対して複数のキーペアが定義される場合は必須です。 |

<!-- markdownlint-enable line-length -->

<Message>

JWTライブラリの使用

独自のJWTへの署名は、複雑で手間のかかる処理になる可能性があります。そのようなことがないよう、事前にこの処理を済ませたライブラリがほぼすべての言語で用意されています。概要については、[JWT.io](https://jwt.io/)をご覧ください。

</Message>

### 4. アクセストークンをリクエストする

The final step is to exchange the short lived JWT assertion for a more long lived Access Token by calling the token endpoint with the assertion as a parameter.

<Tabs>

<Tab title=".Net">

```dotnet
using System.Net;
using System.Net.Http;

var content = new FormUrlEncodedContent(new[]
{
  new KeyValuePair<string, string>(
    "grant_type", "urn:ietf:params:oauth:grant-type:jwt-bearer"),
  new KeyValuePair<string, string>(
    "assertion", assertion),
  new KeyValuePair<string, string>(
    "client_id", config.boxAppSettings.clientID),
  new KeyValuePair<string, string>(
    "client_secret", config.boxAppSettings.clientSecret)
});

var client = new HttpClient();
var response = client.PostAsync(authenticationUrl, content).Result;

class Token
{
  public string access_token { get; set; }
}

var data = response.Content.ReadAsStringAsync().Result;
var token = JsonConvert.DeserializeObject<Token>(data);
var accessToken = token.access_token;
```

</Tab>

<Tab title="Java">

```java
import java.util.ArrayList;
import java.util.List;

import org.apache.http.HttpEntity;
import org.apache.http.NameValuePair;
import org.apache.http.client.entity.UrlEncodedFormEntity;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClientBuilder;
import org.apache.http.message.BasicNameValuePair;
import org.apache.http.util.EntityUtils;

List<NameValuePair> params = new ArrayList<NameValuePair>();

params.add(new BasicNameValuePair(
  "grant_type", "urn:ietf:params:oauth:grant-type:jwt-bearer"));
params.add(new BasicNameValuePair(
  "assertion", assertion));
params.add(new BasicNameValuePair(
  "client_id", config.boxAppSettings.clientID));
params.add(new BasicNameValuePair(
  "client_secret", config.boxAppSettings.clientSecret));

CloseableHttpClient httpClient =
  HttpClientBuilder.create().disableCookieManagement().build();
HttpPost request = new HttpPost(authenticationUrl);
request.setEntity(new UrlEncodedFormEntity(params));
CloseableHttpResponse httpResponse = httpClient.execute(request);
HttpEntity entity = httpResponse.getEntity();
String response = EntityUtils.toString(entity);
httpClient.close();

class Token {
  String access_token;
}

Token token = (Token) gson.fromJson(response, Token.class);
String accessToken = token.access_token;
```

</Tab>

<Tab title="Python 3">

```python
import json
import requests

params = {
    'grant_type': 'urn:ietf:params:oauth:grant-type:jwt-bearer',
    'assertion': assertion,
    'client_id': config['boxAppSettings']['clientID'],
    'client_secret': config['boxAppSettings']['clientSecret']
}
response = requests.post(authentication_url, params)
access_token = response.json()['access_token']
```

</Tab>

<Tab title="Node">

```js
const axios = require('axios')
const querystring = require('querystring');

let accessToken = await axios.post(
  authenticationUrl,
  querystring.stringify({
    grant_type: 'urn:ietf:params:oauth:grant-type:jwt-bearer',
    assertion: assertion,
    client_id: config.boxAppSettings.clientID,
    client_secret: config.boxAppSettings.clientSecret
  })
)
.then(response => response.data.access_token)
```

</Tab>

<Tab title="Ruby">

```ruby
require 'json'
require 'uri'
require 'net/https'

params = URI.encode_www_form({
  grant_type: 'urn:ietf:params:oauth:grant-type:jwt-bearer',
  assertion: assertion,
  client_id: config['boxAppSettings']['clientID'],
  client_secret: config['boxAppSettings']['clientSecret']
})

uri = URI.parse(authentication_url)
http = Net::HTTP.start(uri.host, uri.port, use_ssl: true)
request = Net::HTTP::Post.new(uri.request_uri)
request.body = params
response = http.request(request)

access_token = JSON.parse(response.body)['access_token']
```

</Tab>

<Tab title="PHP">

```php
use GuzzleHttp\Client;

$params = [
  'grant_type' => 'urn:ietf:params:oauth:grant-type:jwt-bearer',
  'assertion' => $assertion,
  'client_id' => $config->boxAppSettings->clientID,
  'client_secret' => $config->boxAppSettings->clientSecret
];

$client = new Client();
$response = $client->request('POST', $authenticationUrl, [
  'form_params' => $params
]);

$data = $response->getBody()->getContents();
$access_token = json_decode($data)->access_token;
```

</Tab>

</Tabs>

## Client credentials grant

Follow the steps below if you would like to verify your application's identity using a client ID and client secret.

### 前提条件

* A Custom Application using Client Credentials Grant authentication within the Box [Developer Console][devconsole]
* [2FA][2fa] enabled on your Box account in order to view/copy your client secret from the configuration tab
* Ensure your application is authorized within the Box Admin Console

<Message notice>

Your client secret is confidential and needs to be protected. Because this is how we securely identify an application's identity when obtaining an Access Token, you do not want to freely distribute a client secret. This includes via email, public forums and code repositories, distributed native applications, or client-side code. 

</Message>

When making your API call to obtain an [Access Token][accesstoken], your request body needs to contain your client ID and client Secret. Set the `grant_type` to `client_credentials`.

If you would like to authenticate as the application's [Service Account][sa]:

* set `box_subject_type` to `enterprise`
* set `box_subject_id` to the enterprise ID

If you would like to authenticate as a Managed User:

* set `box_subject_type` to `user` 
* set `box_subject_id` to the user ID

<Tabs>

<Tab title="cURL">

<!-- markdownlint-disable line-length -->

```cURL
curl --location --request POST ‘https://api.box.com/oauth2/token’ \
--header ‘Content-Type: application/x-www-form-urlencoded’ \
--data-urlencode ‘client_id=<client_id>’ \
--data-urlencode ‘client_secret=<client_secret>’ \
--data-urlencode ‘grant_type=client_credentials’ \
--data-urlencode ‘box_subject_type=enterprise’ \
--data-urlencode ‘box_subject_id=<enterprise_id>’
```

<!-- markdownlint-enable line-length -->

</Tab>

</Tabs>

## コードサンプル

The code in this guide is available on [GitHub][samples].

[samples]: https://github.com/box-community/samples-docs-authenticate-with-jwt-api

[2fa]: https://support.box.com/hc/en-us/articles/360043697154-Two-Factor-Authentication-Set-Up-for-Your-Account

[devconsole]: https://app.box.com/developers/console

[accesstoken]: e://post-oauth2-token/

[sa]: g://authentication/user-types/app-users/#service-accounts

[configfile]: g://applications/custom-apps/jwt-setup/#jwt-keypair

[keypair]: g//authentication/jwt/without-sdk/#public-and-private-keypair

[ccg]: g://authentication/jwt/without-sdk/#client-credentials-grant
