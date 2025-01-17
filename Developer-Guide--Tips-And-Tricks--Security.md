# Security

## About

When developing code for MarkUs, a priority should always be to make the code as secure as possible. As a Ruby on Rails application, MarkUs uses a lot of the built-in security features that Rails offers. Whenever developing code for MarkUs, please read and follow the [Securing Rails Applications guide](https://edgeguides.rubyonrails.org/security.html).

## Developing with Security in Mind

Please see below for MarkUs specific security approaches:

### Cross-Site Request Forgery (CSRF)

- Make sure that `protect_from_forgery with: exception` is set for all routes and the `csrf_meta_tags` are set for all views.

### File Upload

- All files uploaded by users should have their filename and filepath sanitized so that files are only written to disk in approved locations.
- Files should be written to disk with minimal permissions (ie. do not make uploaded files executable!)

### Input Validation

- User input (eg. form fields) should always be validated in both the browser and again when the request is received by the server.
- In the browser, use [client-side form validation](https://developer.mozilla.org/en-US/docs/Learn/Forms/Form_validation).
- Use validations on model attributes on the server side. In some cases, additional input validation may be required in controller methods as well.

### Content Security Policy (CSP) Headers

MarkUs uses content security policies (CSP) help prevent XSS and CSS injection attacks and are defined in `content_security_policy.rb` initializer file using Rails' [CSP DSL](https://edgeguides.rubyonrails.org/security.html#content-security-policy).

In some cases, MarkUs depends on external code that violates it's CSP. In these cases, it is may be necessary to temporarily override these policies for specific routes. Avoid this if at all possible by using dependencies that do not violate the CSPs outlined in `content_security_policy.rb` but if you absolutely have to:

- Only change the policies for the routes that require it. For example, do not change a policy for all routes in a controller if the change is only necessary for one or two routes.
- Change the policies so that they ensure the maximum amount of protection. For example, do not permit content from all https sources if only content from a single url is required.

#### Writing code to comply with CSP

MarkUs' CSP do not permit certain elements to appear in the DOM as they pose a security risk. Please see below for alternative (more secure) ways of writing html and js code:

- Do not include inline style attributes in DOM elements. Make sure all styling is done in css files:

```
// DO NOT DO THIS
<div id="some-id" style="display: none"/>
```

```
// DO THIS

<div id="some-id"/> // in the .html file

#some-id {          // in the .css file
  display: none;
}

```

- Do not include javascript as a string in onclick/onchange/etc. attributes. Instead add event listeners in a javascript file.

```
// DO NOT DO THIS

<a href='#' id="open-something" onclick="openthething(); return false;"/>
```

```
// DO THIS

<a href='#' id="open-something"/>  // in the .html file

$(function() {                    // in the .js file
  $('#open-something').click(function(event) {
    openthething();
    event.preventDefault();
  })
})
```

- Do not add styles in html files using `<style>` tags:

```
// DO NOT DO THIS

<style>             // in an .html file
  #some-id {
    display: none;
  }
</style>
```

```
// DO THIS

#some-id {          // in a .css file
  display: none;
}
```

- Do not add javascript to html files using `<script>` tags:

```
// DO NOT DO THIS

<script>            // in an .html file
  function add_things(x, y) {
    return x + y
  }
</script>
```

Move all javascript into .js files instead and include it in `html.erb` files with `javascript_include_tag nonce: true`. If you absolutely have to include javascript in `<script>` tags, make sure that their nonce value is set:

```
// DO THIS

<%= javascript_tag nonce: true do %>   // in an .html.erb file
  function add_things(x, y) {
    return x + y
  }
<% end %>
```

Nonce values are single-use hashes (they are different for every request) that the browser uses to ensure that the scripts included in the DOM are permitted (not injected by some attacker).


## Security Testing Guidelines

These guidelines are based on the [OWASP Top Ten Web Application Vulnerabilities](http://www.owasp.org/index.php/Top_10_2007). Individual descriptions are followed by a link to the appropriate expanded reference page on the OWASP website.

### A1 - Cross Site Scripting (XSS)

*XSS flaws occur whenever an application takes user supplied data and sends it to a web browser without first validating or encoding that content. XSS allows attackers to execute script in the victim's browser which can hijack user sessions, deface web sites, possibly introduce worms, etc.* ([link](http://www.owasp.org/index.php/Top_10_2007-Cross_Site_Scripting))

Testing for XSS consists of the following:

-   manual testing of individual form fields by entering a simple script such as `<script>window.alert("meow")</script>`
-   manual testing of URLs for parameter-based injection with similar scripts
-   inspection of code to ensure that all views have properly escaped code displayed back to the user
-   scanning with automated tools such as XSS Me

This attack is highly relevant to the MarkUs project, as it can be used to elevate student privileges to TA or admin level.

### A2 - Injection Flaws

*Injection flaws, particularly SQL injection, are common in web applications. Injection occurs when user-supplied data is sent to an interpreter as part of a command or query. The attacker's hostile data tricks the interpreter into executing unintended commands or changing data.* ([link](http://www.owasp.org/index.php/Top_10_2007-Injection_Flaws))

Testing for Injection Flaws consists of the following:

-   manual testing of individual form fields by entering SQL snippets designed to be executed in whichever database the project runs against
-   manual testing of URLs for parameter-based injection with similar scripts
-   inspection of code to ensure that queries are properly parameterized and validated
-   scanning with automated tools such as SQL Inject Me

This attack is highly relevant to the MarkUs project, as it can be used to tamper with or destroy student data.

### A3 - Malicious File Execution

*Code vulnerable to remote file inclusion (RFI) allows attackers to include hostile code and data, resulting in devastating attacks, such as total server compromise. Malicious file execution attacks affect PHP, XML and any framework which accepts filenames or files from users.* ([link](http://www.owasp.org/index.php/Top_10_2007-Malicious_File_Execution))

This is a tricky issue to test for, but the key thing to look for in a code review is proper validation of all filenames submitted to the application by the user, such that the user cannot reference files on the application server.

This is a serious issue in general, but for MarkUs it's somewhat mitigated by the use of Rails.

### A4 - Insecure Direct Object Reference

*A direct object reference occurs when a developer exposes a reference to an internal implementation object, such as a file, directory, database record, or key, as a URL or form parameter. Attackers can manipulate those references to access other objects without authorization.* ([link](http://www.owasp.org/index.php/Top_10_2007-Insecure_Direct_Object_Reference))

The use of MVC frameworks like Rails protects against the most common of DOR attacks involving database records. In regards to testing, OWASP states that "The goal is to verify that the application does not allow direct object references to be manipulated by an attacker." This is done by reviewing the code for instances of direct reference, and manually testing for things like incorrect filesystem access. The classic example here is tampering with a form to request the file `../../../../etc/passwd`.

This is an important area for MarkUs to be tested in, as it involves a lot of files uploaded to the application.

### A5 - Cross Site Request Forgery (CSRF)

*A CSRF attack forces a logged-on victim's browser to send a pre-authenticated request to a vulnerable web application, which then forces the victim's browser to perform a hostile action to the benefit of the attacker. CSRF can be as powerful as the web application that it attacks.* ([link](http://www.owasp.org/index.php/Top_10_2007-Cross_Site_Request_Forgery))

CSRF attacks are easy to test for, but can be challenging to mitigate. Fortunately, Rails has built-in anti-CSRF functionality. See [here](https://guides.rubyonrails.org/security.html#cross-site-request-forgery-csrf) for a writeup.

Testing for CSRF at its most basic involves a code snippet like the following, hosted on another server:

`<img src="http://www.markusproject.org/admin-demo/main/logout"\>`

In an application which checks against CSRF, this will do nothing; in an application where protection is not in place, it will log you out.

CSRF attacks are highly relevant to testing MarkUs, as they allow for privilege escalation and potentially tampering with student data.

### A6 - Information Leakage and Improper Error Handling

*Applications can unintentionally leak information about their configuration, internal workings, or violate privacy through a variety of application problems. Attackers use this weakness to steal sensitive data, or conduct more serious attacks.* ([link](http://www.owasp.org/index.php/Top_10_2007-Information_Leakage_and_Improper_Error_Handling))

Testing for information leakage typically consists of causing the application to error, and examining the result. It is also worth making sure that across the codebase there is a common strategy regarding error handling. Things to look for:

-   Errors from different levels in the application stack: database, framework, web server, your code written on top of the framework
-   Leakage via custom error codes - they may be useful debugging, but what information to they give an attacker?

Tools such as OWASP's WebScarab are ideal for testing error conditions.

This is one of the less serious issues on its own, but information gleaned from overly verbose error messages can be used to exacerbate other attacks.

### A7 - Broken Authentication and Session Management

*Account credentials and session tokens are often not properly protected. Attackers compromise passwords, keys, or authentication tokens to assume other users' identities.* ([link](http://www.owasp.org/index.php/Top_10_2007-Broken_Authentication_and_Session_Management))

Authentication mechanisms should never validate anything on the client side, as an attacker can circumvent such validation. It is advisable to use the built-in session management system for your framework; excercise great caution when extending it or working around it. Things to watch out for and test regarding session management: use of guessable session IDs, failure to correctly expire sessions, and storage of private session variables in the client side browser.

In situations like MarkUs where there is an external auth mechanism in place, it's worth looking at denial of service conditions - what if someone hits the login page with a script repeatedly? Will this crash the login server?

This is a serious issue, but if folks are judicious about sticking with the Rails auth system, there is much less risk.

### A8 - Insecure Cryptographic Storage

*Web applications rarely use cryptographic functions properly to protect data and credentials. Attackers use weakly protected data to conduct identity theft and other crimes, such as credit card fraud.* ([link](http://www.owasp.org/index.php/Top_10_2007-Insecure_Cryptographic_Storage))

MarkUs does not store data encrypted on the disk, but if there is a need to in the future, make sure to use tested and proven cryptographic libraries rather than writing any code which does crypto. Use the correct type of encryption (symmetric, asymmetric, hash) for a particular purpose.

### A9 - Insecure Communications

*Applications frequently fail to encrypt network traffic when it is necessary to protect sensitive communications.* ([link](http://www.owasp.org/index.php/Top_10_2007-Insecure_Communications))

Does your application communicate exclusively over SSL? Is `SESSION_COOKIE_SECURE` set to `True` so that your cookies won't be forced into the clear? What other side channels are used to communicate with your application, and are those secured using standard cryptographic libraries?

The two key takeaways from this issue are:

1.  Use SSL for all sensitive communications with the server, in particular authentication. Use it for everything if you can afford the overhead.

2.  Don't write your own cryptographic functions. Ever.

This is a fairly serious issue for an application which is likely to be use on a hostile network. The UTOR campus wireless is one such network.

### A10 - Failure to Restrict URL Access

*Frequently, an application only protects sensitive functionality by preventing the display of links or URLs to unauthorized users. Attackers can use this weakness to access and perform unauthorized operations by accessing those URLs directly.* ([link](http://www.owasp.org/index.php/Top_10_2007-Failure_to_Restrict_URL_Access))

This is easy to test against. Login as an admin user and copy URLs which only the admin user has access to. Log back in as a non-admin user and try browsing to those URLs. This is known as "forced browsing." If you are able to, there is a problem with the access control mechanisms governing URL access.

This issue is highly relevant to MarkUs due to the possibility of privilege escalation on the part of students.
