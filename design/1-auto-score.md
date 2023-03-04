# Proposal: Automatically score Vulcan assets

Author: Roi Martin

Last updated: 2023-03-03

Discussion at https://github.com/adevinta/graph-proposal/issues/1.

## Abstract

This proposal is a first step towards extending Vulcan to score assets
automatically.
It proposes an implementation that extends the data model of the Vulcan Assets
to include the Blast Radius Score calculated by the Security Graph.
This implementation also covers the services required to keep this score
up-to-date.
It takes into account that, in the future, we will integrate automatic scores
from other data sources.

It is out of the scope of this proposal:
  - Redesigning the current manual scoring implementation used by Vulcan.
  - Extending Vulcan to take automatic actions based on a combination of manual
    and automatic scores.

## Background

Vulcan should be able to automatically take actions based on the specific
characteristics of a given asset.
For instance, there are assets that should be scanned more often due to the
nature of the data they handle or the amount of connections they have with
other systems.
Also, the criticality of a vulnerability or the priority to fix it should not
depend only in the vulnerability itself but also in the context of the affected
assets.

Right now, Vulcan relies on users to manually score their assets.
This data is valuable because there is subjective information that is very
difficult to extract automatically.
However, it is a manual process, which means that the big majority of assets
are not scored and the ones that are quickly become outdated.
Users find also hard to fill this score because because it depends on the
connection of the asset with other assets and their respective scores.

Ideally, assets should be scored automatically.
Now with the Security Graph we have a source of objective data that allows to
automatically characterized assets.
For instance, the Blast Radius Score, calculated using the AWS data contained
in the Security Graph, measures the number of systems that could be potentially
affected by a successful attack.
Of course, there are other useful sources of information and many more
interesting scores.

The final solution must combine manual and automated scores in a single metric
that Vulcan can use to take actions.

## Proposal

### Manual scoring

Let's start by analyzing the current state of asset scoring in Vulcan.
It is part of the `vulcan-api` service.

This is the current definition of the `Asset` type
([source](https://github.com/adevinta/vulcan-api/blob/1faca2de63cd70467e3f1f6b08333cea11fbbe0d/pkg/api/asset.go#L27-L44)):

```go
type Asset struct {
	ID                string             `gorm:"primary_key;AUTO_INCREMENT" json:"id" sql:"DEFAULT:gen_random_uuid()"`
	TeamID            string             `json:"team_id" validate:"required"`
	Team              *Team              `json:"team,omitempty"` // This line is infered from column name "team_id".
	AssetTypeID       string             `json:"asset_type_id" validate:"required"`
	AssetType         *AssetType         `json:"asset_type"` // This line is infered from column name "asset_type_id".
	Identifier        string             `json:"identifier" validate:"required"`
	Alias             string             `json:"alias"`
	Options           *string            `json:"options"`
	EnvironmentalCVSS *string            `json:"environmental_cvss"`
	ROLFP             *ROLFP             `json:"rolfp" sql:"DEFAULT:'R:1/O:1/L:1/F:1/P:1+S:2'"`
	Scannable         *bool              `json:"scannable" gorm:"default:true"`
	AssetGroups       []*AssetGroup      `json:"groups"`      // This line is infered from other tables.
	AssetAnnotations  []*AssetAnnotation `json:"annotations"` // This line is infered from other tables.
	CreatedAt         time.Time          `json:"-"`
	UpdatedAt         time.Time          `json:"-"`
	ClassifiedAt      *time.Time         `json:"classified_at"`
}
```

The `ROLFP` field contains the manual score mentioned in the
[background](#background) section
([source](https://github.com/adevinta/vulcan-api/blob/1faca2de63cd70467e3f1f6b08333cea11fbbe0d/pkg/api/rolfp.go#L16-L26)):

```go
// ROLFP stores the vector containing the dimensions we use to classify the
// impact of an asset.
type ROLFP struct {
	Reputation byte
	Operation  byte
	Legal      byte
	Financial  byte
	Personal   byte
	Scope      byte
	IsEmpty    bool
}
```

It consists on a set of binary indicators that allow to classify the impact of
an asset.
An asset can meet none of the criteria or it can meet multiple criteria.

The `ROLFP` type is persisted in the DB as a string
(e.g. `R:1/O:1/L:0/F:0/P:0+S:1`)
([source](https://github.com/adevinta/vulcan-api/blob/1faca2de63cd70467e3f1f6b08333cea11fbbe0d/pkg/api/rolfp.go#L28-L35)):

```go
// String returns the representation of the ROLFP in the form:
// R:0/O:0/L:0/F:0/P:0+S:0
func (r ROLFP) String() string {
	if r.IsEmpty {
		return ""
	}
	return fmt.Sprintf("R:%d/O:%d/L:%d/F:%d/P:%d+S:%d", r.Reputation, r.Operation, r.Legal, r.Financial, r.Personal, r.Scope)
}
```

This simplifies the REST API, which also uses its string representation
([source](https://github.com/adevinta/vulcan-api/blob/1faca2de63cd70467e3f1f6b08333cea11fbbe0d/docs/swagger.json)):

```json
"AssetPayload": {
  "type": "object",
  "properties": {
    "rolfp": {
      "type": "string",
      "description": "Rolfp plus scope vector",
      "example": "R:1/O:1/L:0/F:0/P:0+S:1"
    }
  }
}
```

The `Scope` field deserves special attention as it modifies the meaning of the
other indicators and it is non-binary.
  - `Scope=0`: The asset **is not** capable of providing access to other
    completely independent assets.
    Thus, the ROLFP takes into consideration only the asset for which the
    assessment is being conducted.
  - `Scope=1`: The asset **is** capable of providing access to other completely
    independent assets and their ROLFP is **known**.
    Thus, The most restrictive value for each of the criteria will be used by
    doing a logical OR (`||`) operation for each criteria and over all of the
    assets, including the original asset for which the assessment is being
    conducted.
  - `Scope=2`: The asset **is** capable of providing access to other completely
    independent assets and their ROLFP is **unknown**.
    Thus, the ROLFP takes into consideration only the asset for which the
    assessment is being conducted.
    However, the actual security impact is unknown.

It is pretty clear that we cannot expect users to deal with this complexity.
The Security Graph contains the relations between assets and should be used to
automate those operations that require working with highly connected data.

Because of how the `Scope` parameter works, the ROLFP value of the asset under
assessment is lost when `Scope` is 1.
We should not lose this data.

The issues explained above this line are signs that the current approach should
be reevaluated.

### Automatic scoring

We propose to add a new optional field named `AutoScore` to the `Asset` type:

```go
type Asset struct {
	// ...
	ROLFP     *ROLFP     `json:"rolfp" sql:"DEFAULT:'R:1/O:1/L:1/F:1/P:1+S:2'"`
	AutoScore *AutoScore `json:"auto_score" sql:"DEFAULT:''"`
	// ...
}
```

The type `AutoScore` contains all the scores that can be updated automatically.
All these scores are optional.

```go
type AutoScore struct {
	BlastRadius *float64
}
```

For the moment, it will only include the Blast Radius Score returned by the
`graph-intel-api` service.
But it is expected to be extended in the future with other automatic scores.

`AutoScore` is stored as a string in the database.
The REST API also uses its string representation.
We don't know yet if this is the right choice.
However, we prefer to start with an implementation that is simple and similar
to ROLFP.
Time will give us a better understanding of the problem and new automatic
scores will be created.
Then we can reevaluate this decision and decide if changes are required.

Similar to `ROLFP`, the marshaled version of `AutoScore` has the following
format:

```
ScoreName1:123.4567/ScoreName2:234.5678/ScoreName3:nil
```

Scores with a `nil` value can be omitted.
Thus, the previous example is equivalent to:

```
ScoreName1:123.4567/ScoreName2:234.5678
```

However, there is a case when they cannot be omitted.
When an asset is updated via REST API, omitted scores are not updated.
This allows to updates scores individually from different services that do not
know each other.
Thus, if a score needs to be set to `nil` it must be explicitly present.

### Automatic score update process

A K8s cronjob will be in charge of updating the `AutoScore` field for every
asset.

The cronjob executes the following steps:

1. Get assets and their corresponding owners using the
   `graph-asset-inventory-api` service.
2. For every asset, get the Blast Radius Score using the `graph-intel-api`
   service.
3. For every asset and owner, update its `AutoScore` field using the
   `vulcan-api` service.

We expect step 2 to be the bottleneck, allowing to update a group of assets
while the next Blast Radius Score is calculated.
If step 3 becomes the bottleneck, we might consider adding a bulk asset update
endpoint to the `vulcan-api`.
Giving that this is a batch process, throughput is not critical.
However, further improvements, like adding a cache layer in front of the
`graph-intel-api` service, will be evaluated if the execution time is not
acceptable.

## Rationale

### Manual/Automatic vs Subjective/Objective

We initially thought on ROLFP as a "subjective score" because it considers
aspects that are hard to evaluate without human input.
Therefore, it would make sense to create an analogous "objective score" that
only takes into consideration facts stored in systems like the Security Graph.
This facts must be based on data that can be automatically gathered and
evaluated.
For instance, the network visibility of an AWS EC2 instance or the permissions
of an AWS IAM role.

However, it is also true that we can use subjective data, like the ROLFP, to
automatically derive an score.
For instance, the ROLFP could be automatically mapped into a priority score.

This proposal approaches score classification from the perspective of manual
and automatic scores.
ROLFP would be an example of manual score because it requires human input to be
generated.
On the other side, the Blast Radius Score is an example of automatic score that
can be automatically generated.

The proposed approach, allows to use the `AutoScore` field to store automatic
scores derived from manual scores.
This way a modification in the ROLFP of an asset could trigger the update of a
specific automatic score.
This could even imply querying the Security Graph to merge multiple ROLFP
scores together, potentially replacing the `Scope` functionality of the current
scoring model.

Finally, it would be the combination of all these automatic scores what is used
to take actions.

### Asset annotations

We cannot know in advance which automatic scores will be assigned to assets.
Therefore, we need to keep flexibility in the way we store them.

The `Asset` field `AssetAnnotations` may look like a good candidate for this
requirement.
However, annotations are basically a key-value store that is freely used to
store metadata.
That means that there is no schema.
Added to that, services writing annotations do not commit to follow a defined
contract.
They can change the name and format of an annotation at their discretion.
Because of all this, services must always treat annotations as optional.

Thus, annotations should be strictly informational and never used in
application logic.

The previous issues discard `AssetAnnotations` as a viable option.

## Implementation

The implementation is fairly short and straightforward.
It does not break backwards compatibility and won't affect the way Vulcan
works right now.
However, these are the foundations for integrating Vulcan and the Security
Graph.
Thus, we need to put special effort on the design phase and document all the
decisions thoroughly.
Roi Martin will do the work during Q1 2023.
