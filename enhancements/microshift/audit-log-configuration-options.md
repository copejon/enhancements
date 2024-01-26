# Configurable Audit Logging for MicroShift

## Metadata

Title: configurable-audit-logging
Authors: Jon Cope jcope@redhat.com
Reviewers: TBD
Approvers: TBD

## Summary

Add ability for MicroShift users to configure API server audit logging policies, storage location, log rotation and retention, and actions when disk capacity reached.

## Motivation

MicroShift currently uses a hardcoded audit logging policy. It should be configurable in a manner similar to OpenShift to meet customer needs.

### User Stories

* As a MicroShift administrator, I want to configure audit logging policies so that I can control what events are logged.

* As a MicroShift administrator, I want to specify a custom log file location so that I can write audit logs to a dedicated volume or device.

* As a MicroShift administrator, I want to configure the max file size and retention policy for audit logs so that I can better manage their disk usage.

* As a MicroShift administrator managing devices with limited storage capacity, I want to configure fall-back behaviors for logs should the device reach a storage limit I specify or the disk reach capacity.

### Goals and Non-Goals

**Goals:**

- Enable MicroShift administrators to manage logging policies as a tiered set of profiles.
- Provide flexibility to specify storage location, log sizes and retention policies.

**Non-Goals:**

- Custom Rules: MicroShift is a single-user system without user groups, so custom rules similar to OpenShift are not required and should be explicitly marked as out of scope.

## Proposal

### User Workflow

1. Administrator edits MicroShift config file to specify desired audit logging policy profile
2. Administrator edits MicroShift config file to specify audit log location if required
3. Administrator edits MicroShift config file to specify max file size and retention policy for logs
4. Administrator selects action to take when storage limit or capacity reached
5. MicroShift service is restarted to apply changes

### API Extensions

**Audit Log Policy Configuration:**

- MicroShift config’s `apiServer` root-field must include a child node for storing log configurations.

```yaml
apiServer:
  auditLog: MAP(STRING)INTERFACE
```

- The `apiServer` field must include a child node for storing the audit log policy profile to use. Policies and descriptions are listed below. These are the same policies used by OpenShift.

```yaml
apiServer:
  auditLog:
    policy: STRING
```

| Policy             | Description                                                                                                                                                                                                                           |
|--------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Default            | Logs only metadata for read and write requests; does not log request bodies except for OAuth access token requests. This is the default policy.                                                                                       |
| WriteRequestBodies | In addition to logging metadata for all requests, logs request bodies for every write request to the API servers (create, update, patch, delete, deletecollection). This profile has more resource overhead than the Default profile. |
| AllRequestBodiesIn | addition to logging metadata for all requests, logs request bodies for every read and write request to the API servers (get, list, create, update, patch). This profile has the most resource overhead.                               |
| None               | No requests are logged; even OAuth access token requests and OAuth authorize token requests are not logged.                                                                                                                           |

**Audit Log File Storage Location:**

- `path` field to specify non-default audit log file paths.  Accepted values must be absolute paths.

```yaml
apiServer:
  auditLog:
    path: STRING
```

**Audit Log File Rotation:**

- `maxFileSize` config variable to set maximum audit log file size. Accepted values must be a string with a number and unit suffix with no separator character, e.g. `10GB`. Valid units are "KB", "MB", "GB", "TB", "KiB", "MiB", "GiB", and "TiB".

```yaml
apiServer:
  auditLog:
    maxFileSize: STRING
```

**Audit Log Retention Policy:**

- `retainPolicy` variable for system response when disk space full
- Supported values: "ignore", "rotate", "terminate"

| Policy      | Description                                             |
|-------------|---------------------------------------------------------|
| `ignore`    | Do nothing.                                             |
| `rotate`    | Trigger the specified rotation behavior                 |
| `terminate` | Kill the MicroShift process to prevent loss of log data |

**Logging:**

- Log audit log policy additions, removals, changes
- Log audit log rotations and locations

### Implementation Details

* All audit logging configuration fields must be optional.  If not specified, the default audit logging policy will be used.
* There are three services in MicroShift that generate audit logs: The `kube-apiserver`, 
* MicroShift will use the `logrotate` utility to rotate audit logs. Policies must be internally implemented as `logrotate` configurations.
* The log rotate service must be a dependency of the MicroShift systemd service so that it is consistently available when MicroShift is running. 
* If not specified, audit logging policy must default to "Default"
* Logging policies are executed on MicroShift startup.  Changes to the policy will not take effect until MicroShift is restarted. 
* Add ability to specify custom audit log file location
* Add config parameters for max audit log file size and retention policy


## Design Details

- SOS reports collect audit logs as part of the report.  If a users specifies a log location other than the default, the SOS report must include the custom location.

### Open Questions

- Custom log paths: Can sos collect reports from dynamic paths? If not, can we symlink the log directory to as a means of faking a static path while writing logs to a custom location?  If so, can sos follow these symlinks?
- OVK audit log configurations are stored in a configMap. Because it's run in a container, logs are stored under /var/cont

### Test Plan

* Unit tests to validate new config APIs
* Integration tests to verify configured audit logging policies are applied properly

### Graduation Criteria

* Upstream code, tests, docs merged
* Downstream builds, docs updated
* Automated CI coverage for new functionality
* QE test plans defined

#### Dev Preview -> Tech Preview

- Ability to utilize the enhancement end to end
- End user documentation completed and published
- Sufficient test coverage
- Gather feedback from users
- Available by default

#### Tech Preview -> GA

- Sufficient time for feedback
- End-to-end tests

## Implementation History

* Audit logging config added upstream
* Adapted for MicroShift implementation
* Downstream packages and docs updated

## Alternatives

* Continue using hardcoded audit logging policy
* Limited subset of OpenShift's configuration options

## Infrastructure Needed