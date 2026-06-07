# Web Cache Deception Checklist

## Cache Discovery

[ ] X-Cache header
[ ] Age header
[ ] CF-Cache-Status
[ ] Akamai indicators
[ ] Fastly indicators

## Dynamic Endpoints

[ ] Profile
[ ] Orders
[ ] Invoices
[ ] Notifications
[ ] Dashboard
[ ] Documents

## Static Extension Rules

[ ] .css
[ ] .js
[ ] .ico
[ ] .png
[ ] .jpg
[ ] .svg
[ ] .exe

## Delimiters

[ ] ;
[ ] .
[ ] ,
[ ] :
[ ] !
[ ] |
[ ] $

## Encoded Delimiters

[ ] %23
[ ] %3f
[ ] %00
[ ] %09
[ ] %0a

## Static Directories

[ ] /static
[ ] /assets
[ ] /images
[ ] /js
[ ] /css

## Exact Match Files

[ ] favicon.ico
[ ] robots.txt
[ ] index.html

## Exploit Validation

[ ] Cache hit confirmed
[ ] Sensitive data present
[ ] Cross-user exposure
[ ] Reproducible
