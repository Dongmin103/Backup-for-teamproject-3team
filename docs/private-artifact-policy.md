# Private Artifact Policy

The following are private/off-Git only:

- DB dumps and database exports
- raw datasets and PHI-like data
- Lambda zip packages
- Docker image tarballs
- SageMaker/model artifacts
- OpenSearch/Neptune exports if they contain data
- API keys, credentials, passwords, secret values

Store private artifacts outside Git or under an ignored path such as `.aws-exit-backup/private/`.
Secrets and PHI-like data must be encrypted. If a non-secret/non-PHI artifact cannot be encrypted, record the artifact class, reason, owner, location, ACL/access limits, retention window, compensating controls, and remediation date.
