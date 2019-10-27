# conformix
Extracts testing directives from web server configuration files, executes specified tests and analyzes results.

When configuring your webservers you need a simple method to test the validity of configuration. Using `conformix` you may
define testing urls inside your configuration files and later run the tool to check whether all testing directives pass.

`conformix` will scan your files passed as arguments, detect urls to curl and what to look for inside the server responses
to consider tests as passed or failed. Testing directives are simply transformed to series of `curl` and `grep` calls,
with a possibility to provide command-line parameters for both.

## Usage
```
./conformix /etc/nginx/sites-available/*.conf
```

Configuration files must contain `@test` and `@test-result` directives in comments around normal configuration directives,
which represent the curl commands to execute and specific patterns to look for inside the curl responses.

## Test directives format
```
@test [CURL_ARGS] URL
where
    CURL_ARGS list of arguments to pass to curl
    URL       url to test with curl call

@test-result [GREP_ARGS] PATTERN
where
    GREP_ARGS list of arguments to pass to grep
    PATTERN   grep pattern that must appear in curl response (extended regexp)
```

`@test` directives are transformed to `curl -s` calls and their responses are cached. All the following `@test-result`
directives are checked against this response. Similar `@test` directives reuse already existing response from the
previous `curl` call.

`@test-result` directives are transformed to `egrep` calls and are executed on the last response received from the
previous `@test` directive. If several `@test-result` directives are present after a `@test` they will be tested in
order of appearance over the same `curl` response.

Each `grep` invocation must return at least one line, in which case the corresponding test is considered as passed. If
`grep` returns empty string the corresponding test is considered failed.

In case of failed tests the dump of failed tests will be displayed.

Same `@test` directives will reuse the result returned by first invocation. To illustrate the concept:
```
@test http://www.mysite.com/
  -> will run curl -s http://www.mysite.com/ command and store the result in (a)
@test -I http://www.mysite.com/page.php
  -> will run curl -s -I http://www.mysite.com/page.php command and store the result in (b)
@test http://www.mysite.com/
  -> will not run curl but use the result stored at (a)
```

## Example:

### http://www.mysite.com/ must redirect to HTTPS:
- 301 status is set
- `Location` header contains `https:` redirect

```
# @test -I http://www.mysite.com/
# @test-result '^HTTP.+ 301 '
# @test-result '^Location: https://www\.mysite\.com/'
```

This test will issue `curl -s -I http://www.mysite.com/` command (only http headers with empty body). The response will
be checked first with `egrep '^HTTP.+ 301 '` and then with `egrep '^Location: https://www\.mysite\.com/'` commands.

### https://www.mysite.com/ must be of UTF-8 mime type
- `Content-Type` header contains `UTF-8`

```
# @test -I https://www.mysite.com/
# @test-result -i '^content-type: .+utf-8'
```

This test will issue `curl -s -I https://www.mysite.com/` command. The response will be checked with
`egrep -i '^content-type:.+utf-8'` command (case insensitive).
