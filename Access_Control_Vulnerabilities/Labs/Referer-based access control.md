
1. Log in with admin credentials
2. Downgrade or upgrade any user;
3. Send the request from step `2` to repeater;

![](PortSwigger-Solution/static/img/Pasted_image_20231120104837.png)
4. Login in with `wiener` and copy the session cookie;
5. Change the session cookie from step `3` to cookie copied from  step `4`;

![](PortSwigger-Solution/static/img/Pasted_image_20231120105032.png)

![](PortSwigger-Solution/static/img/Pasted_image_20231120105039.png)

> Note,  if you change the `Referer` header path to anything than /admin you cant resolve this lab ;)