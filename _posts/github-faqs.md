---
layout: post
title: GitHub FAQs
categories: [GitHub]
use_math: true
keywords: github, faq
---



## 連線相關問題

### WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!

[參考來源](https://blog.faq-book.com/?p=3826)

以下是出現的錯誤訊息：

<section style = "background:#000; padding:5%; color:#fff;display:inline-block" >
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the RSA key sent by the remote host is
SHA256: {YOUR_SHA_CODE}.
Please contact your system administrator.
Add correct host key in /Users/tim/.ssh/known_hosts to get rid of this message.
Offending RSA key in /Users/tim/.ssh/known_hosts:1
Host key for github.com has changed and you have requested strict checking.
Host key verification failed.
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
</section>

<br><br>

從訊息中可以知道，主要問題出在 **known_hosts** 中使用了錯誤的 **RSA key** 。 我們有兩種做法：

1. 將 **known_hosts** 檔案或內部相關的 Key 給刪除。
   像是 GitHub 的 key 就會寫成：
   ```
   github.com ssh-ed25519 {ED25519_KEY}
   github.com ssh-rsa {RSA_KEY}
   github.com ecdsa-sha2-nistp256 {SHA2_NISTP256_KEY}
   ```
2. 將 **known_hosts** 移除，砍掉重練。

若是砍掉重練，我們需要進行以下步驟：
1. 建立 key
   ```shell
   $ ssh-keygen -t {FILE_NAME} -C "{YOUR_ACCOUNT_EMAIL}"
   ```
2. 將 key 放入 [GitHub](https://github.com/settings/ssh) 中
3. 回到 terminal 執行 push

此時你可能會看到：
```shell
The authenticity of host 'github.com (20.27.177.113)' can't be established.
ED25519 key fingerprint is SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

記得要與 [GitHub Doc](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints) 中的公開 fingerprints 比對以下，確保 fingerprint 正確。

然後打 `yes` 即可。


當然，還有另一個方法。 我們只需要在建立 key 時寫：
```shell
ssh-keygen -R {IP ie 192.168.2.151}
```


<br><br><br><br><br>
