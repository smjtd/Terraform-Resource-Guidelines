# Contributing to Terraform - IBM Cloud Provider

Some guidelines that we expect from our contributors are as follows:
<!-- TOC depthFrom:2 -->

- [Pull Requests](#pull-requests)
    - [Pull Request Lifecycle](#pull-request-lifecycle)
    - [Checklists for Contribution](#checklists-for-contribution)



<!-- /TOC -->

## Pull Requests

Below are some guidelines to follow before Raising a PR. 
### Pull Request Lifecycle
1. Fork the repository, add your code changes. You can raise a PR for review and comments.
2. One team member from terraform-provider-ibm will take a look at the PR, will either approve it  or give comments if anything more needed to be done. 
3. Once all the comments and checklists are done, your contributions will be merged into the codebase. Contributions that are merged will be included into the next release of the provider.
Checklist

Before raising a PR for a new resource on the IBM Cloud Provider, please ensure the following:

- [ ]  Did you follow all the guidance, while  [designing a Resource & Datasource for IBM Cloud Provider](#Designing-a-Resource-&-Datasource-for-IBM-Cloud-Provider) ?
- [ ]  Did you follow all the guidance, while implementing the resource & data sources ?
- [ ]  All the Resources & Data sources have Unit-test. See [Writing Acceptance
   Tests](#writing-acceptance-tests) below for a detailed guide on how to
   approach these.
- [ ]  All the Resources & Data sources have the required Documentation, with examples
- [ ]  All the Resources & Data sources are included in a sample terraform configuration & published in the /examples folder

### Checklists for Contribution

If you are contributing for a new terraform resource or a datasource ensure that you fulfil all checklist items-
#### Pre-requisites
 If you are trying to create resource for a new IBM cloud service you should have a golang sdk for the respective service.
-  Review the Services API (swagger or openapi definition) & Services CLI.
-  Review the Services Go-Lang SDK (open-sourced), and try the examples in the SDK

	-   If the SDK is not ready, use [openapi-sdkgen tool ](https://github.ibm.com/CloudEngineering/openapi-sdkgen) to generate the golang sdk
	-   publish the [go-lang SDK]([https://github.com/IBM-Cloud/bluemix-go](https://github.com/IBM-Cloud/bluemix-go)) in opensource (preferably, in this location - http://github.com/IBM-Cloud/
#### Designing a Resource & Datasource for IBM Cloud Provider
-  Design the schema for the Resource & Datasource that will include the attributes based on the payloads field in the API. Use minimalist approach;
-   Review the cli functionality of the service to determine the required and optional arguments.
- Specify force new to the attribute if that attribute is not responsible for updating the resource on successive apply .
- Review the response payload and determine the computed fields.
- In Schema, if an attribute name is more than one word separate word with _ (underscore).
#### Implementing the Resource code for IBM Cloud Provider service
- In the provider codebase vendor the latest sdk in using **go mod vendor**
-  Initialize the client on config.go.
	-  Import the sdk and write init functions.
	- map to iam authorisations wherever required. Eg with resource_group resource in **config.go**.
```go
package ibm
//import the sdk after vendoring
import "github.com/IBM-Cloud/bluemix-go/api/resource/resourcev2/managementv2"
//initialize the function inside ClientSession interface
type  ClientSession  interface {
ResourceManagementAPIv2() (managementv2.ResourceManagementAPIv2, error)
}
//initialize the error variables and service instance inside ClientSession interface
type  clientSession  struct {
resourceManagementConfigErrv2 error
resourceManagementServiceAPIv2 managementv2.ResourceManagementAPIv2
}

// Define the ResourceManagementAPIv2 function...
func (sess clientSession) ResourceManagementAPIv2() (managementv2.ResourceManagementAPIv2, error) {
return sess.resourceManagementServiceAPIv2, sess.resourceManagementConfigErrv2

// ClientSession configures and returns a fully initialized ClientSession
func (c *Config) ClientSession() (interface{}, error) {
resourceManagementAPIv2, err := managementv2.New(sess.BluemixSession)
if err != nil {
	session.resourceManagementConfigErrv2 = fmt.Errorf("Error occured while 							configuring Resource Management service: %q", err)
	}
session.resourceManagementServiceAPIv2 = resourceManagementAPIv2
}
```
If any service requires iam oauth-token for client configuration map with 
```go 
sess.BluemixSession.Config.IAMAccessToken
```
- Define the schema and CRUD operations based on api support, and write the function logic and call the respective sdk functions for crud instances. Each new resource will go in a separate file eg. resoure_ibm_resource_name.go
- Include timeout block based on the time required for create, update, delete respectively.
-  Add wait logic if required to read the status availability of the resource, check status from api response payload if supported.
- In Read method set all values defined in schema.
- Add Exist and import functionality  (only for resource).
- Write Update function based on api support.
- In Delete function at end setID(") to null. Eg take resource_group  **resource_ibm_resource_group.go**
```go
package ibm
//import the sdk and terraform libraries
import (
    "fmt"
    "github.com/IBM-Cloud/bluemix-go/api/resource/resourcev2/managementv2"
    "github.com/IBM-Cloud/bluemix-go/bmxerror"
    "github.com/IBM-Cloud/bluemix-go/models"
    "github.com/hashicorp/terraform/helper/schema"
)
//Define the function for the resource that contains schema and logic for CRUD instances
func  resourceIBMResourceGroup() *schema.Resource {

//Schema definition looks like
	return &schema.Resource{
		Create: resourceIBMResourceGroupCreate,
		Read: resourceIBMResourceGroupRead,
		Update: resourceIBMResourceGroupUpdate,
		Delete: resourceIBMResourceGroupDelete,
		Exists: resourceIBMResourceGroupExists,
		Importer: &schema.ResourceImporter{},
		Schema: map[string]*schema.Schema{
			"name": {
			Type: schema.TypeString,
			Required: true,
			Description: "The name of the resource group",
			},

			"quota_id": {
			Type: schema.TypeString,
			Optional: true,
			Description: "The id of the quota",
			Removed: "This field is removed",
			},

			"default": {
			Description: "Specifies whether its default resource group or not",
			Type: schema.TypeBool,
			Computed: true,
			},

			"state": {
			Type: schema.TypeString,
			Description: "State of the resource group",
			Computed: true,
			},

			"tags": {
			Type: schema.TypeSet,
			Optional: true,
			Elem: &schema.Schema{Type: schema.TypeString},
			Set: schema.HashString,
			},
		},
	}
}

  
//Function for create
func  resourceIBMResourceGroupCreate(d *schema.ResourceData, meta interface{}) error {
rMgtClient, err := meta.(ClientSession).ResourceManagementAPIv2()

if err != nil {
    return err
}
name := d.Get("name").(string)
userDetails, err := meta.(ClientSession).BluemixUserDetails()

if err != nil {
    return err
}

accountID := userDetails.userAccount
resourceGroupCreate := models.ResourceGroupv2{
ResourceGroup: models.ResourceGroup{
Name: name,
AccountID: accountID,
},

}
resourceGroup, err := rMgtClient.ResourceGroup().Create(resourceGroupCreate)

if err != nil {
    return fmt.Errorf("Error creating resource group: %s", err)
}
d.SetId(resourceGroup.ID)
return  resourceIBMResourceGroupRead(d, meta)
}

//Function for read feature
func  resourceIBMResourceGroupRead(d *schema.ResourceData, meta interface{}) error {
rMgtClient, err := meta.(ClientSession).ResourceManagementAPIv2()

if err != nil {
    return err
}
resourceGroupID := d.Id()
resourceGroup, err := rMgtClient.ResourceGroup().Get(resourceGroupID)

if err != nil {
    return fmt.Errorf("Error retrieving resource group: %s", err)
}


d.Set("name", resourceGroup.Name)
d.Set("state", resourceGroup.State)
d.Set("default", resourceGroup.Default)
return  nil
} 

//Function for update feature
func  resourceIBMResourceGroupUpdate(d *schema.ResourceData, meta interface{}) error {

rMgtClient, err := meta.(ClientSession).ResourceManagementAPIv2()

if err != nil {
    return err
}
resourceGroupID := d.Id()
updateReq := managementv2.ResourceGroupUpdateRequest{}
hasChange := false

if d.HasChange("name") {
    updateReq.Name = d.Get("name").(string)
    hasChange = true
}

if hasChange {
    _, err := rMgtClient.ResourceGroup().Update(resourceGroupID, &updateReq)
    if err != nil {
        return fmt.Errorf("Error updating resource group: %s", err)
    }
}

return  resourceIBMResourceGroupRead(d, meta)
}

//Function for delete feature
func  resourceIBMResourceGroupDelete(d *schema.ResourceData, meta interface{}) error {
rMgtClient, err := meta.(ClientSession).ResourceManagementAPIv2()

if err != nil {
    return err
}
resourceGroupID := d.Id()
err = rMgtClient.ResourceGroup().Delete(resourceGroupID)

if err != nil {
    return fmt.Errorf("Error Deleting resource group: %s", err)
}

//set ID to null (mandatory)
d.SetId("")
return  nil
}

//Function for exist feature
func  resourceIBMResourceGroupExists(d *schema.ResourceData, meta interface{}) (bool, error) {
rMgtClient, err := meta.(ClientSession).ResourceManagementAPIv2()\
if err != nil {
    return  false, err
}

resourceGroupID := d.Id()
resourceGroup, err := rMgtClient.ResourceGroup().Get(resourceGroupID)

if err != nil {
    if  apiErr, ok := err.(bmxerror.RequestFailure); ok {
        if apiErr.StatusCode() == 404 {
            return  false, nil
        }
    }

    return  false, fmt.Errorf("Error communicating with the API: %s", err)
}

return resourceGroup.ID == resourceGroupID, nil
}
```
put error handling and proper status messages wherever required.