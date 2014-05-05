How To Use Nginx as Global Traffic Director
-------------------------------------------

> Nginx, DNS

Global Traffic Directing is a way to redirect users to the closest geographically located server.
This way users from Asia would be served by servers in Asia and users in Europe by servers in Europe.

Most if not all major DNS providers support this feature, but it sometimes is too expensive.
This tutorial will explain how you can setup "poor man's gtd" using Nginx, GeoIP and Subdomains.
