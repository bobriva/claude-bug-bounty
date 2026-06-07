# API Security Knowledge

## Core Principle

The API is often larger than the web application that consumes it.

Many endpoints:

- Are undocumented
- Are legacy
- Are mobile-only
- Are internal-only

Yet remain accessible.

## High-Risk Areas

1. Hidden Endpoints

2. Hidden Parameters

3. Mass Assignment

4. Authorization Failures

5. Excessive Data Exposure

6. Business Logic Flaws

## Mindset

Never trust the frontend to reveal the full API attack surface.

Always enumerate:

- Endpoints
- Methods
- Parameters
- Versions
- Content Types