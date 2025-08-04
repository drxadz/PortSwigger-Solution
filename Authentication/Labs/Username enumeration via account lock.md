

- Burp configuration payloads
	- Cluster bomb
		- Payload 1, with  usernames wordlists
![](PortSwigger-Solution/static/img/Pasted_image_20231106201404.png)
- Payload 2, set to null payload with length 5

![](PortSwigger-Solution/static/img/Pasted_image_20231106201446.png)
![](PortSwigger-Solution/static/img/Pasted_image_20231106201617.png)
- This username is the "one" that trigger the error message, "too many attempts".
- Retry the attack with, against this username with the password list, from the lab
![](PortSwigger-Solution/static/img/Pasted_image_20231106201958.png)
- Look, there is no error message with **ad** user and the **password** 112233
- Wait a bit and login with that 

![](PortSwigger-Solution/static/img/Pasted_image_20231106202142.png)
Lab solved!