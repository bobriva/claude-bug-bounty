# SQL Injection Checklist

## Discovery

[ ] GET parameters
[ ] POST parameters
[ ] JSON body
[ ] XML body
[ ] Cookies
[ ] Headers

## Error-Based

[ ] Single quote test
[ ] Double quote test
[ ] SQL error observed

## Boolean-Based

[ ] True condition
[ ] False condition
[ ] Response difference

## UNION

[ ] Column count identified
[ ] UNION accepted
[ ] Data extracted

## Blind

[ ] Boolean inference
[ ] Time delay inference
[ ] Character extraction possible

## OAST

[ ] DNS interaction
[ ] HTTP interaction

## Impact

[ ] Sensitive data exposed
[ ] Authentication bypass
[ ] Data modification possible
[ ] Server compromise potential
