---
layout: post
title:  "Google Play : Uploading App"
date:   2022-08-08 16:15:07 +0800
categories: [android, google play]
---

### Background
In this post, we will learn step by step process to upload your app to Google Play.


### Keystore

#### Creating and Managing



#### Github

```Groovy
def keystorePropertiesFile = rootProject.file("keystore.properties")
def keystoreProperties = new Properties()
keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
```


```Groovy
storePassword = my.keystore
keyPassword   = key_password
keyAlias      = my_key_alias
storeFile     = store_file
```
