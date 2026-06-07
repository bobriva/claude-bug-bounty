# NoSQL Injection Knowledge

## Definition

NoSQL Injection occurs when user-controlled input alters the logic of NoSQL database queries.

## Major Categories

1. Syntax Injection

2. Operator Injection

3. JavaScript Injection

4. Timing-Based Injection

## Common Targets

MongoDB

CouchDB

Firebase

DynamoDB

## High Risk Operators

$where

$regex

$ne

$in

$exists

## Common Impacts

Authentication Bypass

Data Disclosure

Privilege Escalation

Field Enumeration

Blind Data Extraction