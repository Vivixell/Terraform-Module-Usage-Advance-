# Advanced Terraform Module Usage: Versioning, Gotchas, and Reuse Across Environments

![releases](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/tyvfgltz5f67ikxm0ups.png)

If you followed my Day 8 breakdown, you know that moving from a monolithic `main.tf` to a reusable Terraform module is a massive architectural leap. But building a module is only half the battle. 

If you don't understand how Terraform resolves paths, handles state logic, or pins versions, your beautiful module will quickly become a nightmare for other engineers to use. 

For Day 9 of the 30-Day Terraform Challenge, I am diving deep into enterprise module management. We are going to cover the three most common module "gotchas" that break deployments, how to properly version your code using Git, and the exact multi-environment pattern Platform Teams use to test new infrastructure safely.

---

## Part 1: The 3 Terraform Module Gotchas

When you transition from writing root environments to writing child modules, Terraform's behavior changes in subtle ways. Here are three mistakes that are incredibly easy to make and deeply frustrating to debug.

### Gotcha 1: The File Path Trap
Imagine you have a `user-data.sh` script sitting in your module folder. You might be tempted to reference it like this:

```json
# ❌ THE WRONG WAY

resource "aws_launch_template"  "app" {

  # ...

  user_data = filebase64("./user-data.sh") 
}
```

**Why it fails:** Terraform resolves the `./` relative to the directory where you run `terraform apply` (the root environment), not where the module lives. When your Prod environment calls this module, Terraform will look for `prod/user-data.sh` and crash.


**The Fix:** Always use the `path.module` expression. This tells Terraform to dynamically locate the script based on the module's actual location.

#### ✅ THE RIGHT WAY

```json

resource "aws_launch_template" "app" {
  
  user_data = filebase64("${path.module}/user-data.sh") 
}

```
Other possible values are:
| Name | Description |
|---|:---|
|`path.module`| The directory where the current module's code is located. (Use this 99% of the time inside modules to reference scripts or templates).|
|`path.root` |The directory of the root environment (e.g., your dev/ or prod/ folder where you actually run terraform apply).|
|`path.cwd` |The directory from which the user executed the Terraform command in their terminal. (Usually identical to path.root, unless the user runs Terraform from a different directory using the -chdir flag).|

### Gotcha 2: The Inline Block "Perpetual Diff"

Some AWS resources, like Security Groups, allow you to define rules inline or as separate resources. When writing modules, inline blocks are dangerous.

#### ❌ THE WRONG WAY (Inline)

```json
resource "aws_security_group" "web" {
  ingress {
    from_port = 80
    to_port   = 80
    protocol  = "tcp"
  }
}
```

**Why it fails:** If a developer calls your module and later decides they need to attach a custom VPN rule to that same Security Group using an external aws_security_group_rule, Terraform will panic. The module claims strict ownership of the inline block and will constantly try to delete the developer's new rule, resulting in an endless loop of infrastructure changes.


**The Fix:** Always use standalone resources inside modules to allow external extensibility.

#### ✅ THE RIGHT WAY (Standalone)
```json

resource "aws_security_group" "web" {
  name = "web-sg"
}

resource "aws_security_group_rule" "http" {
  type              = "ingress"
  security_group_id = aws_security_group.web.id
  from_port         = 80
  to_port           = 80
  protocol          = "tcp"
}

```

### Gotcha 3: The Blunt depends_on

Sometimes, a developer calling your module will try to force an execution order using depends_on.

#### ❌ THE WRONG WAY
```json

module "webserver" {
  source     = "../modules/webserver"
  depends_on = [aws_database_instance.main]
}
```
**Why it fails:** This forces Terraform to treat your entire webserver cluster as a single, opaque block. If anything changes in the database (even just updating a tag), Terraform might mistakenly taint the entire module and attempt to destroy and recreate your Auto Scaling Group and Load Balancer.

**The Fix:** Let Terraform's native dependency graph do the work. Pass explicit resource outputs into the module as variables.

#### ✅ THE RIGHT WAY
```json


module "webserver" {
  source     = "../modules/webserver"
  db_address = aws_database_instance.main.address # Implicit dependency created safely!
}

```
## Part 2: Module Versioning & Source Syntax

In a production environment, you never point your infrastructure at a local folder. You point it at a version-controlled registry. This guarantees that a change to the module code doesn't instantly break every environment using it.

To version a module, you simply use Git tags.

```json
# Tagging a stable release in your module repository
git tag -a "v0.0.1" -m "Initial stable release"
git push origin v0.0.1
```
Once tagged, how you format the `source` URL determines exactly what Terraform pulls down. Here is what the syntax looks like across different ecosystems:

### 1. The Local Source (Good for initial drafting, bad for teams):

```json
module "webserver" {
  source = "../modules/webserver"
}
```
### 2. The Git Repository Source (The Enterprise Standard):
Notice the double slash `//.` This tells Terraform where the repository ends and the specific subfolder begins, while `?ref=` targets the exact Git tag.

```bash
module "webserver" {
  source = github.com/Vivixell/Reusable-Infrastructure//modules/webserver?ref=v0.0.1"
}
```
### 3. The Terraform Public Registry Source:
When pulling from HashiCorp's official registry, the syntax simplifies. The version gets its own explicit argument.

```json
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
}
```
## Part 3: The Multi-Environment Deployment Pattern

Now that our module is versioned remotely, we can implement the true Platform Engineering lifecycle: **Dev tests the new features, Prod stays pinned to stability.**

I released `v0.0.1` of [my webserver module](https://github.com/Vivixell/Reusable-Infrastructure/tree/master/modules/webserver), and then added a new `custom_tags` feature and released `v0.0.2`. Here is how my root environments consume them simultaneously without conflict:

#### The Production Environment (Pinned to Stable)
Production code should never use the master branch or the "latest" tag. It is strictly pinned to the battle-tested `v0.0.1` release.

```bash
# prod/main.tf
module "webserver_cluster" {
  source = github.com/Vivixell/Reusable-Infrastructure//modules/webserver?ref=v0.0.1"

  cluster_name  = "prod-app"
  instance_type = "t3.small"
  # ... standard inputs ( check the repo for the complete code)
}
```
[check the repo for the complete code](https://github.com/Vivixell/Terraform-Module-Usage-Advance-)

#### The Development Environment (Testing Bleeding Edge)

The Dev team is actively testing the new tagging feature. Their environment points to `v0.0.2`.

```json
# dev/main.tf
module "webserver_cluster" {
  source = "[github.com/Vivixell/Reusable-Infrastructure//modules/webserver?ref=v0.0.2](https://github.com/Vivixell/Reusable-Infrastructure//modules/webserver?ref=v0.0.2)"

  cluster_name  = "dev-app"
  instance_type = "t3.micro"
  
  # Testing the new feature introduced in v0.0.2!
  custom_tags = {
    Environment = "Development"
    Owner       = "OVR"
  }
}
```

When you run `terraform init` in these separate folders, Terraform downloads the exact, isolated versions of the code. If `v0.0.2` has a catastrophic bug, Production remains completely safe.

## Final Thoughts

Understanding file paths, inline blocks, and versioning is what separates writing Terraform scripts from building actual Infrastructure as Code architecture.

