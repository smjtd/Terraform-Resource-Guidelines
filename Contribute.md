# Contributing to Terraform - IBM Cloud Provider

Some guidelines that we expect from our contributors are as follows:
<!-- TOC depthFrom:2 -->

- [Pull Requests](#pull-requests)
    - [Lifecycle](#lifecycle)
    - [Guidelines for Contribution](#Guidelines-for-contribution)
        - [Pre-requisites](#Pre-requisites) 
        - [Designing a Resource & Datasource for IBM Cloud Provider](#Designing-a-Resource-&-Datasource-for-IBM-Cloud-Provider) 
        - [Implementing the Resource code for IBM Cloud Provider service](#Implementing-the-Resource-code-for-IBM-Cloud-Provider-service) 
        - [Writing Acceptance Tests](#Writing-acceptance-tests)
        - [Updating Documentation](#Updating-Documentation)
        - [Adding examples](#Adding-Examples)


<!-- /TOC -->

## Pull Requests

Below are some guidelines to follow before Raising a PR. 
### Lifecycle
1. Fork the repository, add your code changes. You can raise a PR for review and comments.
2. One team member from terraform-provider-ibm will take a look at the PR, will either approve it  or give comments if anything more needed to be done. 
3. Once all the comments and checklists are done, your contributions will be merged into the codebase. Contributions that are merged will be included into the next release of the provider.
Checklist

Before raising a PR for a new resource on the IBM Cloud Provider, please ensure the following:

- [ ]  Did you follow all the guidance, while  [designing a Resource & Datasource for IBM Cloud Provider](#Designing-a-Resource-&-Datasource-for-IBM-Cloud-Provider) ?
- [ ]  Did you follow all the guidance, while [implementing the resource & data sources](#Implementing-the-Resource-code-for-IBM-Cloud-Provider-service) ?
- [ ]  All the Resources & Data sources have Unit-test. See [Writing Acceptance
   Tests](#Writing-acceptance-tests) below for a detailed guide on how to
   approach these.
- [ ]  All the Resources & Data sources have the required [Documentation](#Updating-Documentation), with examples
- [ ]  All the Resources & Data sources are included in a [sample terraform configuration](#Adding-examples) & published in the /examples folder

### Guidelines for Contribution

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

- In **Provider.go** add the entry to the resource name and map it to the respective function (follow naming standards) Eg.
```go
    ResourcesMap: map[string]*schema.Resource{
        "ibm_resource_group": resourceIBMResourceGroup(),
    }
```


#### Writting Writing Acceptance Tests
- Write Basic unit test configuration covering cases for all scenarios and follow these guidelines for writing acceptance tests .
1. Unit test for each resource should go to separate file inside ibm/ directory named similar to the actual resource file followed by _test. Eg. resource_file_name_test.go
2. Use **CheckDestroy** function- CheckDestroy typically named testAccCheckIBMResourceABCDestroy is called after all test steps have been run and Terraform has run `destroy` on the remaining state. This allows developers to ensure any resource created is truly destroyed. This method receives the last known Terraform state as input, and commonly uses infrastructure SDKs to query APIs directly to verify the expected objects are no longer found, and should return an error if any resources remain. 
4. Use **testAccPreCheck** function - **PreCheck** if non-nil, will be called before any test steps are executed. It is commonly used to verify that required values exist for testing, such as environment variables containing test keys that are used to configure the Provider or Resource under test.
5. Use **fmt.Sprintf()** for defining testcase scenarios in hcl format.
6. Use randomised naming for creating new resources that should generate a unique nave every time you run the test case.
eg, **resource_ibm_resource_group_test.go**

```go
//Configuration for testing a scenario
func TestAccIBMResourceGroup_Basic(t *testing.T) {
	var conf models.ResourceGroupv2
	resourceGroupName := fmt.Sprintf("terraform_%d", acctest.RandInt())
	resourceGroupUpdateName := fmt.Sprintf("terraform_%d", acctest.RandInt())

	resource.Test(t, resource.TestCase{
		PreCheck:  func() { testAccPreCheck(t) },
		Providers: testAccProviders,
		Steps: []resource.TestStep{
			resource.TestStep{
				Config: testAccCheckIBMResourceGroup_basic(resourceGroupName),
				Check: resource.ComposeAggregateTestCheckFunc(
					testAccCheckIBMResourceGroupExists("ibm_resource_group.resourceGroup", &conf),
					resource.TestCheckResourceAttr("ibm_resource_group.resourceGroup", "name", resourceGroupName),
					resource.TestCheckResourceAttr("ibm_resource_group.resourceGroup", "default", "false"),
					resource.TestCheckResourceAttr("ibm_resource_group.resourceGroup", "state", "ACTIVE"),
				),
            },
            resource.TestStep{
				ResourceName:      "ibm_resource_group.resourceGroup",
				ImportState:       true,
				ImportStateVerify: true,
			},
		},
	})
}
//check exist function
func testAccCheckIBMResourceGroupExists(n string, obj *models.ResourceGroupv2) resource.TestCheckFunc {

	return func(s *terraform.State) error {
		rs, ok := s.RootModule().Resources[n]
		if !ok {
			return fmt.Errorf("Not found: %s", n)
		}

		rsContClient, err := testAccProvider.Meta().(ClientSession).ResourceManagementAPIv2()
		if err != nil {
			return err
		}
		resourceGroupID := rs.Primary.ID

		resourceGroup, err := rsContClient.ResourceGroup().Get(resourceGroupID)
		if err != nil {
			return err
		}

		obj = resourceGroup
		return nil
	}
}

//checkDestroy function
func testAccCheckIBMResourceGroupDestroy(s *terraform.State) error {
	rsContClient, err := testAccProvider.Meta().(ClientSession).ResourceManagementAPIv2()
	if err != nil {
		return err
	}

	for _, rs := range s.RootModule().Resources {
		if rs.Type != "ibm_resource_group" {
			continue
		}

		resourceGroupID := rs.Primary.ID

		// Try to find the key
		_, err := rsContClient.ResourceGroup().Get(resourceGroupID)

		if err != nil && !strings.Contains(err.Error(), "404") {
			return fmt.Errorf("Error waiting for resource group (%s) to be destroyed: %s", rs.Primary.ID, err)
		}
	}

	return nil
}

//test case scenario
func testAccCheckIBMResourceGroup_basic(resourceGroupName string) string {
	return fmt.Sprintf(`
		  
		  resource "ibm_resource_group" "resourceGroup" {
			name     = "%s"
		  }
	
```

Finally to run the test files use make test

#### Updating Documentation
- Documentation for each resource goes into separate .html.markdown file under website directory. put the files inside directories /d for datasource or /r for resource.
- Describe the functionality of the resource along with all the supported arguments and attributes exported.
- Add a small example in the example section of the documentation file. 

#### Adding examples
- Add an example .tf configuration for any use case for the service in the /examples folder.
- Use proper standards, define variables inside variable.tf, initialise the provider in provider.tf and write the configuration in main.tf
