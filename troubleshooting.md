# Troubleshooting

Draft of upstream docs for improving debug experience:
* https://github.com/konveyor/enhancements/pull/2

Work In Progress for a flowchart
* [MTC Debug flowchart](https://app.lucidchart.com/documents/view/d0907ce1-ccf1-4226-86eb-e5332f9d42a4/0_0)

# Generating a must-gather
`oc adm must-gather --image=registry.redhat.io/rhcam-1-2/openshift-migration-must-gather-rhel8`

# How to Clean Up/Reset a migration
## Failed migration, how to clean up and retry
* Ensure stage pods have been cleaned up.  If a migration fails during stage or copy, the 'stage' pods will be retained to allow debugging.  Before proceeding with a reattempt of the migration the stage pods need to be manually removed.

## Scale a quiesced application back

## Note on labels applied to help track what was migrated



