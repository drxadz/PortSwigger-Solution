
1. Try to access the admin `panel`;
2. Verify that, you can't;

![](PortSwigger-Solution/static/img/Pasted_image_20231120102452.png)

3. Remove the `/admin` path and add the header `X-Original-Url: /admin`
4. Verify that, now you can access;
![](PortSwigger-Solution/static/img/Pasted_image_20231120102738.png)
5. Verify the content of the page, and the app only accepts POST request to delete a user;

![](PortSwigger-Solution/static/img/Pasted_image_20231120102918.png)


![](PortSwigger-Solution/static/img/Pasted_image_20231120102925.png)
