**EKS upgrade Process**:

Step 1: ldentify the cluster needs to be upgraded and make a note of all the pods running in diferent
namespace.
Step 2: we will be using highlander upgrade where we are first going to create a eks with latest version
then we will connect with the team as per step1 to get their resources migrated to the new eks and then
destroy the same.

Step 3: In order to upgrade we need to update our terraform base code first using this repo
https://bitbucketdc.abcde.net/projects/CTSBUILDTOOLS/repos/90333-102198-terraform-
base/browse?at=refs%2Fheads%2Ffeature%2FGTS-513162
Where we need to update the module version as per latest terraform bundle.
Along with this we need to update aws stack pave version which can be found in terraform base where
we have to update the latest version including new tf bundles and manually deploy it in 90333-cloudops
ws below are the details. We need to update it as per migration strategy document.
$ tfil configure
Address> https://tfe.abcde.net
Org> 90333-102198-DEV
Project Default Project
