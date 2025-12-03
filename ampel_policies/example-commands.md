# Being familiar with GH API

Traditionally, [branch protection rules](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/managing-a-branch-protection-rule) were defined in a monolithic way.

Then, a more flexible alternative was available via the definition of [rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets).

There is no right or wrong. Maintainers decide what is the best approach for them while both are valid.

These are the two API entrypoints to check these settings and be more familiar with the results:
```bash
ORG="marcusburghardt"
REPO="github-standards"
gh api --method GET /repos/$ORG/$REPO/branches/main
gh api --method GET /repos/$ORG/$REPO/rules/branches/main
```

# Collect data with snappy

[snappy](https://github.com/carabiner-dev/snappy) is a nice tool that allow us to collect data from an API with some additional features to specify filters and attest the responses.

Here is an example on how to use it:
```bash
snappy snap builtin:github/branch-rules.yaml -v ORG=$ORG -v REPO=$REPO -v BRANCH=main --attest > branch-rules.intoto.json
```

# Sign the statement
The snappy attested data can be signed with [bnd](https://github.com/carabiner-dev/bnd) to create a bundle.

```bash
bnd statement branch-rules.intoto.json --out branch-rules.bundle.json
```

# Send to GH (needs permission)
[bnd](https://github.com/carabiner-dev/bnd) also helps to upload the attestation artifact to GH.

It could also be done with [action/attest](https://github.com/marcusburghardt/github-standards/pull/6) and some other variations, but it using `bnd` is much simpler.

```bash
bnd push marcusburghardt/github-standards branch-rules.bundle.json
```

After this, you should already be able to see the attestations in you repo. e.g.:
- https://github.com/marcusburghardt/github-standards/attestations

# Pack the bundles
In case we have more attestation artifacts, they could be packed in a single bundle. But here we have only one.

```bash
bnd pack --bundle branch-rules.bundle.json > attestation.jsonl
```

# Verify Attestation

Now is the nice part. Let's download this attestation and verify it.
I am using a specific [RUN](https://github.com/marcusburghardt/github-standards/attestations/14363049) you can choose any attestation to reproduce.

```bash
ATT_FILE="github-standards-attestations-14363049-sigstore.json"
curl -o $ATT_FILE https://github.com/marcusburghardt/github-standards/attestations/14363049/download
```

## Inspect

```bash
bnd inspect $ATT_FILE
```

## Verify Identity
```bash
bnd verify $ATT_FILE --identity https://github.com/marcusburghardt/github-standards/.github/workflows/compliance.yml@refs/heads/main --issuer https://token.actions.githubusercontent.com
```

Or this identity check could also be skipped in some test environments
```bash
bnd verify $ATT_FILE --skip-identity
```

## Extract data to explore
```bash
bnd extract predicate $ATT_FILE > predicate.json
bnd extract statement $ATT_FILE > statement.json
```

# Check Compliance

Once you validated and attestation and are familiar with their predicates, we can use an Ampel policy to check compliance.

```bash
ampel verify --policy ampel_policies/branch-protection-rules.json --attestation $ATT_FILE \
--subject-hash sha256:3e1416d7258b308178d26d80ee05d989df46e35336ef0d8dcce4516d93a764f8
```

# Examples
Here are some examples on this implemented in CI:
- https://github.com/marcusburghardt/github-standards/actions/runs/19877666241
- https://github.com/marcusburghardt/github-standards/actions/workflows/compliance.yml

# References
Some documentation were put together here:
- https://github.com/marcusburghardt/github-standards/pull/9
