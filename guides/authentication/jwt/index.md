---
rank: 2
related_endpoints: []
related_guides:
  - applications/select
  - authentication/user-types
  - authentication/select
required_guides:
  - authentication/select
related_resources: []
alias_paths:
  - /docs/authenticate-with-jwt
category_id: authentication
subcategory_id: authentication/jwt
is_index: true
id: authentication/jwt
type: guide
total_steps: 4
sibling_id: authentication
parent_id: authentication
next_page_id: authentication/jwt/as-user
previous_page_id: authentication/jwt/with-sdk
source_url: >-
  https://github.com/box/developer.box.com/blob/main/content/guides/authentication/jwt/index.md
fullyTranslated: true
---
# JWT認証

Server-side authentication using JSON Web Tokens (JWT) is the most common way to authenticate to the Box API. JWT is an [open standard](https://jwt.io/) designed to allow powerful server-to-server authentication.

<ImageFrame border>

![JWTフロー](./jwt-flow.png)

</ImageFrame>

Server-side authentication using JWT is only available to the Custom Application [app type][app-type]. This authentication method does not require end-user interaction and, if granted the proper privileges, can be used to act on behalf of any user in an enterprise.

There are two ways you can verify an application's permissions:

* using a public and private key pair
* using a client id and client secret (Client Credentials Grant)

To learn more about these options visit our guide on using [JWT without SDKs][jwtnosdk]. 

<Message warning>

At this time, our SDKs do not support the Client Credential Grant.

</Message>

Upon authorizing a JWT application in the Box Admin Console, a [Service Account][user-types] is automatically generated and is the default Access Token used when authenticating. This is an admin-like user and why applications leveraging JWT require explicit Box Admin approval before use.

## JWTを使用する場合

JWTを使用するサーバー側認証は、以下に当てはまるアプリに最適な認証方式です。

* Boxアカウントを持たないユーザーを使用する
* 独自のIDシステムを使用する
* ユーザーにBoxを使用していることを認識させたくない
* アプリケーションのBoxアカウントにデータを保存し、ユーザーのBoxアカウントには保存しない

[app-type]: g://applications/select/

[user-types]: g://authentication/user-types

[jwtnosdk]: g://authentication/jwt/without-sdk/
