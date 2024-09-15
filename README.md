# hcledit: A NodeJS Wrapper for HCL Document Manipulation
`hcledit` is a versatile Node.js library that facilitates the manipulation of `HashiCorp Configuration Language (HCL)` documents with ease. It serves as a wrapper around [hcledit](https://github.com/mercari/hcledit/tree/main) and the [hashicorp/hcl](https://github.com/hashicorp/hcl) package, offering powerful capabilities for working with HCL configurations using a query and selector syntax reminiscent of [jq](https://github.com/jqlang/jq).

**NOTE: THIS PACKAGE HAS CURRENTLY NO SUPPORT FOR WINDOWS.**

## Key features
* **NodeJS Compatibility**: `hcledit` is designed to seamlessly integrate with NodeJS, making it a versatile choice for developers working within the NodeJS ecosystem.

* **No Go Runtime Required**: Unlike some other HCL manipulation tools, hcledit does not rely on the Go runtime environment. Instead, it leverages [WebAssembly](https://webassembly.org/) to expose a fully compatible Node.js interface.
* **Plug and play**: It also includes options to `serialize` and `deserialize` variables that start with `data`, `local`, and `var` by `stringifying` or `desterilizing` them, ensuring data integrity during the conversion process. You can also pass your own serialize functions.

With `hcledit`, you can effortlessly perform tasks such as updating existing HCL configurations, extracting specific data from HCL documents, and executing complex transformations.

## Installation
To get started, install `hcledit` using npm or yarn:

```bash
npm install @datails/hcledit
# or
yarn add @datails/hcledit
```

## Usage Examples

### Parsing HCL to JSON
To ensure the integrity of your HCL configuration, variables that start with `data`, `local`, and `var` will be stringified before parsing. You can disable this behavior by passing options.

```typescript
import { toJSON, SerializeOptions, createSerializeOpts } from '@datails/hcledit';

const hcl = `
  pull_request_health_check "pull_request" {
    metric_factor "age" {
      metric_name = "PR age"
      scale {
        from = 72
        to = 24
      }
    }

    metric_factor "activity" {
      metric_name = local.activity_name
      scale {
        from = var.activity_scale.from
        to = var.activity_scale.to
      }
    }
  }
`;

const json = await toJSON(hcl); // => { pull_request_health_check: [ ... ] }

// using custom serialize fn
const json2 = await toJSON(obj, createSerializeOpts({
  serialize: (str: string) => str
})); 
```

### Converting JSON to HCL
Converts JSON back to HCL.
```typescript
import { toHCL, DeserializeOptions, createDeserializeOpts } from '@datails/hcledit';

const obj = {
  module: {
    s3: {
      foo: 1,
      bar: 2
    }
  }
};


const hcl = await toHCL(obj); // => '"module" "s3" {\n  "foo" = 1\n\n  "bar" = 2\n}'

// using custom deserialize fn
const hcl2 = await toHCL(obj, createDeserializeOpts({
  deserialize: (str: string) => str
})); 
```

#### Options
The library provides options to control variable handling during conversion:

* `SerializeOptions`:
  * `serializeVariables` (default: `true`): Set to false to disable variable stringification.
  * `serialize` (default: `addQuotesAroundVariables`): Custom function for variable stringification.
* `DeserializeOptions`:
  * `deserializeVariables` (default: `true`): Set to false to disable variable deserialization.
  * `deserialize` (default: `removeQuotesAroundVariables`): Custom function for variable deserialization.

#### Custom Variable Handling
If you need custom variable handling, you can create your own serialization and deserialization functions. The provided default functions are `addQuotesAroundVariables` for serialization and `removeQuotesAroundVariables` for deserialization. You can replace these functions with your own logic to handle variable stringification and deserialization according to your specific requirements.

---

### Working with HCL Files
Given the following HCL configuration file we want to manipulate:

```hcl
module "s3_bucket" {
  source = "terraform-aws-modules/s3-bucket/aws"

  bucket = "my-s3-bucket"
  acl    = "private"

  control_object_ownership = true
  object_ownership         = "ObjectWriter"

  versioning = {
    enabled = true
  }
}
```

Updating an Attribute:

```typescript
import { createHclFileParser } from '@datails/hcledit'

const { updateAttr } = await createHclFileParser()

await updateAttr('example.tf', 'module.s3_bucket.acl', 'public')
  .catch(console.error) // => returns true or false
```

```diff
module "s3_bucket" {
  source = "terraform-aws-modules/s3-bucket/aws"

  bucket = "my-s3-bucket"
+ acl    = "public"

  control_object_ownership = true
  object_ownership         = "ObjectWriter"

  versioning = {
    enabled = true
  }
}
```

Creating an attribute:

```typescript
import { createHclFileParser } from '@datails/hcledit'

const { createAttr } = await createHclFileParser()

await createAttr('example.tf', 'module.s3_bucket.versioning.foo', 'bar')
  .catch(console.error) // => returns true or false
```

```diff
module "s3_bucket" {
  source = "terraform-aws-modules/s3-bucket/aws"

  bucket = "my-s3-bucket"
  acl    = "public"

  control_object_ownership = true
  object_ownership         = "ObjectWriter"

  versioning = {
    enabled = true
+   foo     = bar
  }
}
```

Deleting an attribute:

```typescript
import { createHclFileParser } from '@datails/hcledit'

const { delAttr } = await createHclFileParser()

await delAttr('example.tf', 'module.s3_bucket.object_ownership')
  .catch(console.error) // => returns true or false
```

```diff
module "s3_bucket" {
  source = "terraform-aws-modules/s3-bucket/aws"

  bucket = "my-s3-bucket"
  acl    = "public"

  control_object_ownership = true
- object_ownership         = "ObjectWriter"

  versioning = {
    enabled = true
  }
}
```

Deleting a resource block:

```typescript
import { createHclFileParser } from '@datails/hcledit'

const { delBlock } = await createHclFileParser()

await delBlock('example.tf', 'module.s3_bucket')
  .catch(console.error) // => returns true or false
```

```diff
- module "s3_bucket" {
-  source = "terraform-aws-modules/s3-bucket/aws"

-  bucket = "my-s3-bucket"
-  acl    = "public"

-  control_object_ownership = true
- object_ownership         = "ObjectWriter"

-  versioning = {
-    enabled = true
-  }
- }
```

Renaming a resource block:

```typescript
import { createHclFileParser } from '@datails/hcledit'

const { renameResourceBlock } = await createHclFileParser()

await renameResourceBlock('example.tf', 'module.s3_bucket', 'module.s3_bucket_1')
  .catch(console.error) // => returns true or false
```

```diff
+ module "s3_bucket_1" {
  source = "terraform-aws-modules/s3-bucket/aws"

  bucket = "my-s3-bucket"
  acl    = "public"

  control_object_ownership = true
  object_ownership         = "ObjectWriter"

  versioning = {
    enabled = true
  }
 }
```

Getting an attribute:

```typescript
import { createHclFileParser } from '@datails/hcledit'

const { getAttr } = await createHclFileParser()

const value = await getAttr('example.tf', 'module.s3_bucket.object_ownership')
  .catch(console.error) // => returns true or false
```

Wildcard search:

```typescript
import { createHclFileParser } from '@datails/hcledit'

const { getAttr } = await createHclFileParser()

const value = await getAttr('example.tf', 'module.*.object_ownership') // using a * as wildcard could return an array of values
  .catch(console.error) // => returns true or false
```

Getting keys with a specific value:

```typescript
import { createHclFileParser } from '@datails/hcledit'

const { getKeys } = await createHclFileParser()

const value = await getAttr('example.tf', 'module.*.object_ownership')
  .catch(console.error) // => ["module.s3_bucket.object_ownership"]object_ownership exists.
```

---

### Working with HCL Content
Updating an Attribute in a String:


```typescript
import { createHclContentParser } from '@datails/hcledit'

const { updateAttr } = await createHclContentParser()

const newContent = await updateAttr(`
module "s3_bucket" {
  source = "terraform-aws-modules/s3-bucket/aws"

  bucket = "my-s3-bucket"
  acl    = "private"

  control_object_ownership = true
  object_ownership         = "ObjectWriter"

  versioning = {
    enabled = true
  }
}
`, 'module.s3_bucket.acl', 'public') // => updated acl attribute to public
```

Creating a New Attribute in a String:

```typescript
import { createHclContentParser } from '@datails/hcledit'

const { createAttr } = await createHclContentParser()

const newContent = await createAttr(`
module "s3_bucket" {
  source = "terraform-aws-modules/s3-bucket/aws"

  bucket = "my-s3-bucket"
  acl    = "private"

  control_object_ownership = true
  object_ownership         = "ObjectWriter"

  versioning = {
    enabled = true
  }
}
`, 'module.s3_bucket.versioning.foo', 'bar') // => added foo = bar to versioning object
```

Deleting an Attribute in a String:
```typescript
import { createHclContentParser } from '@datails/hcledit'

const { delAttr } = await createHclContentParser()

const newContent = await delAttr(`
module "s3_bucket" {
  source = "terraform-aws-modules/s3-bucket/aws"

  bucket = "my-s3-bucket"
  acl    = "private"

  control_object_ownership = true
  object_ownership         = "ObjectWriter"

  versioning = {
    enabled = true
  }
}
`, 'module.s3_bucket.object_ownership') // => removed object_ownership property
```

Deleting a resource block:

```typescript
import { createHclContentParser } from '@datails/hcledit'

const { delBlock } = await createHclContentParser()

const newContent = await delBlock(`
module "s3_bucket" {
  source = "terraform-aws-modules/s3-bucket/aws"

  bucket = "my-s3-bucket"
  acl    = "private"

  control_object_ownership = true
  object_ownership         = "ObjectWriter"

  versioning = {
    enabled = true
  }
}
`, 'module.s3_bucket') // => removed module s3_bucket
```

Renaming a resource block:

```typescript
import { createHclContentParser } from '@datails/hcledit'

const { renameResourceBlock } = await createHclContentParser()

const newContent = await renameResourceBlock(`
module "s3_bucket" {
  source = "terraform-aws-modules/s3-bucket/aws"

  bucket = "my-s3-bucket"
  acl    = "private"

  control_object_ownership = true
  object_ownership         = "ObjectWriter"

  versioning = {
    enabled = true
  }
}
`, 'module.s3_bucket', 'module.s3_bucket_1') // => renamed module s3_bucket
```

Getting an attribute:

```typescript
import { createHclContentParser } from '@datails/hcledit'

const { getAttr } = await createHclContentParser()

const value = await getAttr(`
module "s3_bucket" {
  source = "terraform-aws-modules/s3-bucket/aws"

  bucket = "my-s3-bucket"
  acl    = "private"

  control_object_ownership = true
  object_ownership         = "ObjectWriter"

  versioning = {
    enabled = true
  }
}
`, 'module.s3_bucket.object_ownership') // => true
  .catch(console.error)
```

Getting keys with a specific value:

```typescript
import { createHclFileParser } from '@datails/hcledit'

const { getKeys } = await createHclFileParser()

const value = await getAttr(`
module "s3_bucket" {
  source = "terraform-aws-modules/s3-bucket/aws"

  bucket = "my-s3-bucket"
  acl    = "private"

  control_object_ownership = true
  object_ownership         = "ObjectWriter"

  versioning = {
    enabled = true
  }
}
`, 'module.*.object_ownership')
  .catch(console.error) // => ["module.s3_bucket.object_ownership"]
```

When an attribute is not found, `null` is returned.

## Contributing to the Project
We welcome contributions from the community and are excited to see how you can improve this project! If you have a bug fix, feature addition, or any other improvement, please follow these steps to contribute:

1. **Fork and Create a Branch:** Start by forking the repository and then create a separate branch for your contribution. Please name your branch in a way that reflects the nature of your changes.

2. **Describe Your Changes:** When committing your changes, provide a descriptive commit message that gives an overview of what you've done. If you're fixing a bug or adding a feature, briefly explain the problem or the feature.

3. **Update Documentation:** If your changes require it, update the documentation accordingly. This helps ensure that everyone can understand and use the new features or fixes.

4. **Ensure Compatibility with `Go 1.18:** This project utilizes WebAssembly (WASM), and it's crucial that the WASM is generated using Go version 1.18. Please ensure that your contributions are compatible with this version.

5. **Test Your Changes:** Before submitting your pull request, thoroughly test your changes to ensure they work as expected and do not introduce new bugs.

6. **Submit a Pull Request:** Once you have tested your changes and are satisfied with them, submit a pull request to the main repository. In your pull request description, include a detailed explanation of your changes and the reason behind them.

7. **Code Review:** After submitting, your pull request will undergo a review process. Be open to feedback and be ready to make necessary revisions based on suggestions from the repository maintainers.

8. **Merge:** Once your pull request is approved, it will be merged into the main branch.

Thank you for considering contributing to our project! Your efforts help make our software better for everyone.

## License
`hcledit` is open-source software licensed under the `MIT` License. See the [LICENSE](https://gitlab.com/datails/node-hcl-edit/-/blob/master/LICENCE) file for details.

## Acknowledgments
We would like to express our gratitude to the authors and contributors of [hcledit](https://github.com/mercari/hcledit/tree/main) and [hashicorp/hcl](https://github.com/hashicorp/hcl) for their valuable work, which made this project possible.

## Contact
For any questions or feedback, please reach out to us at [info@datails.nl](mailto:info@datails.nl).