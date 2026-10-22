# MariaDB major version upgrades

This chart keeps MariaDB data in the StatefulSet's `data` volume. Change one
image version at a time, let the StatefulSet replace its pod, then validate the
database and application before moving to the next version. The chart enables
`MARIADB_AUTO_UPGRADE` by default so the official image runs `mariadb-upgrade`
before starting MariaDB when a system-table upgrade is needed. A startup probe
allows that work to finish before liveness checks begin.

## Suggested checkpoint path from 10.4

Use the latest patch in each checkpoint series and review that version's
release notes before upgrading. The following checkpoints are a conservative
path through supported LTS series; MariaDB also documents direct upgrades from
any earlier release, so the intermediate steps are operational validation
checkpoints rather than a vendor requirement.

| From | To | Example image tag |
| --- | --- | --- |
| 10.4 | 10.5 | `10.5.28` |
| 10.5 | 10.6 | `10.6.28` |
| 10.6 | 10.11 | `10.11.19` |
| 10.11 | 11.4 | `11.4.13` |
| 11.4 | 11.8 | `11.8.9` |
| 11.8 | 12.3 | `12.3.3` |

The examples reflect releases available when this chart version was prepared.
Choose a later stable patch in a series when one is available. For an upgrade
to a different target series, set that version explicitly for the hop.
MariaDB 10.5 and 10.6 have reached community end of life; treat them as brief
transition checkpoints and continue promptly to 10.11 or later.

## Procedure for every hop

1. Take and verify a restorable backup of all databases. Keep a storage-level
   snapshot if your storage provider supports one. The image's automatic
   upgrade backup covers only the `mysql` system database.
2. Review the release notes between the current and target versions, including
   removed options, storage engine support, and application compatibility.
3. Update the release to the next version, pinning the image tag:

   ```sh
   helm upgrade <release> jbk8s/mariadb \
     --namespace <namespace> --reuse-values \
     --set existingSecret=mariadb-secrets \
     --set upgrade.autoUpgrade=true \
     --set startupProbe.enabled=true \
     --set-string image.tag=10.5.28
   ```

   Replace `10.5.28` with the next checkpoint in the table. Keep the default
   `upgrade.autoUpgrade=true` for the rollout.
4. Wait for the StatefulSet pod to become Ready, inspect its logs for
   `mariadb-upgrade` completion or errors, and check the server version:

   ```sh
   kubectl rollout status statefulset/<statefulset> -n <namespace>
   kubectl logs statefulset/<statefulset> -n <namespace>
   kubectl exec -n <namespace> statefulset/<statefulset> -- sh -ec 'mariadb -uroot -p"$MYSQL_ROOT_PASSWORD" -Nse "SELECT VERSION()"'
   helm test <release> --namespace <namespace>
   ```

   The SQL command reads the root password from the running container. The
   Helm test connects to the configured database using the application user
   and password from the configured Secret, then prints the server version.
5. Validate the application against this version and review database and
   application logs. Continue only when both pass.
6. Repeat for the next checkpoint. Do not use `helm rollback` to downgrade a
   database after it has opened the upgraded data directory. Restore the
   pre-upgrade backup or storage snapshot if recovery requires the old version.

`image.tag` defaults to this chart's `appVersion` (`12.3.3` in this release).
Pin it to the current database version before the first upgrade if installing
this chart would otherwise jump past the first checkpoint. When the chart
manages its Secret, upgrades retain generated password values already present
in that Secret instead of generating replacements.

For deployments whose credentials are supplied by External Secrets, set
`existingSecret` to the ExternalSecret target name. In this environment that
target is `mariadb-secrets`, with keys `mariadb-root-password`,
`mariadb-db-user`, `mariadb-db-password`, and `mariadb-db`. The Helm test uses
the application credentials, avoiding remote root authentication, which may
be restricted to localhost.

## References

- [MariaDB: Upgrading between major versions](https://mariadb.com/docs/server/server-management/install-and-upgrade-mariadb/upgrading/platform-specific-upgrade-guides/upgrading-on-linux/upgrading-between-major-mariadb-versions)
- [MariaDB: mariadb-upgrade](https://mariadb.com/docs/server/clients-and-utilities/deployment-tools/mariadb-upgrade)
- [MariaDB Docker image: `MARIADB_AUTO_UPGRADE`](https://mariadb.com/docs/server/server-management/automated-mariadb-deployment-and-administration/docker-and-mariadb/mariadb-server-docker-official-image-environment-variables)
