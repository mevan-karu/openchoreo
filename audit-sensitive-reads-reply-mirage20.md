> AFAIK, the API does not return the Vault version (I am assuming you are referring to the Vault version here). It is going to be the latest, since GetSecret and ListSecrets don't take a version. If that is the case, will there be any benefit of recording the version in the secret read audits? Maybe we can log with a keyed hash? We only need to identify the value content, right?

The version I meant was the Kubernetes `resourceVersion` of the Secret on the target plane, which is where `GetSecret` and `ListSecrets` read the values from. But you're right, it can't be traced back to the content.

A keyed hash is the better option for identifying the content, but I don't think we need it now. Secret creates and updates are already audited, so a read can be placed against the updates around it by time. A keyed hash also brings a key to manage, and rotating that key breaks the link between old records and the values they hashed.

So I'll drop the version from secret reads. We can add keyed hashes later if identifying the value that was read becomes a requirement.

> Regarding the search text in QueryLogs and GetPlatformLogs, is there any benefit to recording it? I am wondering whether it could have side effects, like unintentionally recording PII or other sensitive data that someone searched for, such as a customer email or a leaked token. These audit records are going to be retained longer than the logs, right?

The audit record should say what was actually read, and for a log query the search text is part of that. Without it, the record shows that someone queried production logs, but not that they searched them for `password`. Splunk and BigQuery record the query text for the same reason.

Some PII in the audit trail is expected. Every record already carries the actor's identity and source IP. Reading the trail needs its own `auditlogs:view` permission, and we'll document the search text as personal data in the audit schema reference.

On retention, yes, audit records are kept longer than the logs, 365 days by default. That's normal for an audit trail, since it has to outlive the data it describes.
