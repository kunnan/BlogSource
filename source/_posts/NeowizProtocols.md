---
title: Neowiz Protocols
date: 2017-11-08 17:54:55
tags: MUG
---

For all Neowiz games. A similar API protocol is used.
<!-- more -->
## Basics
The only variation is the difference between API endpoints. Every request is a HTTP POST request to the API endpoint.The HTTP body is a json serialized string of the request.
The header also requires the following fields

  - Api-Token : *Pass empty string at first.Will be returned by corresponding API later*
  - Fp : *The MD5 Hash of (SecretKey+HTTPBody)*
  - Nce : *Random 32characters alphanumerical string*
  - Secret-Key : *Pass empty string at first.Will be returned by corresponding API later*
  - Secret-Ver : *Pass 1 at first.Will be returned by corresponding API later*
  - X-Unity-Version : *Game's version. Hardcoded in the binary*

## HTTPBody
Each item in the HTTP body consists of the request's id ,the method's name as well as params.  
For example:
``
[{"id":9,"method":"user.loginV2","params":["ACCESS_TOKEN"," ","","iOS","CN"]}]
``  
contains only one request, which has a id of 9 and name "user.loginV2".
There are 5 parameters in this request, being `"ACCESS_TOKEN"," ","","iOS","CN"`

## Response
The response is also a json serialized array containing the responses to the request sorted in the same order as the request array.  
For example:  

``
[
	{
		"result": {
			"API_TOKEN": "API_TOKEN",
			"SECRET_KEY": "DMQGLBlive7",
			"SECRET_VER": "1",
			"guid": "11111",
			"recom_code": "213SDADd",
			"displayName": FOO",
			"profileImg": "http://img.pmangplus.com/members/09332449/profile_img",
			"INTRO_SERVER": "https://dmqglb.mb.pmang.com/DMQ/rpc"
		},
		"error": null,
		"id": 9
	}
]
``
Contains one response, corresponding to the previous request


## Methods
The available methods of each game and their arguments can be found by decompiling the game's Unity DLL or do a HTTP packet capture.
