## AWS Transit Gateway Attachment Automation

This repo contains the files that assist in the automation of Transit Gateway and Transit Gateway Attachment associations for multiple account within an AWS Organization. In addition to resource creation, an AWS Lambda function is included that creates a default route to the created Transit Gateway based upon a user set Tag value. 

## Solution Overview

As IT environments grow, they can become more complex, with additional accounts, VPCs, and the networking between them. <a href="https://aws.amazon.com/transit-gateway/">AWS Transit Gateway</a> is a service that addresses networking complexity by building a hub-and-spoke network to simplify your network routing and security. With Transit Gateway, you can connect your <a href="https://aws.amazon.com/vpc/">virtual private clouds</a> (VPCs) that span multiple accounts and on-premises networks to a single gateway.</p>
<p>While Transit Gateway eases network administration and complexity, it does not address typical environment missteps such as configuration drift and multiple worker administration.</p>
<p>Automation through Infrastructure as Code helps combat deployment inconsistencies and assist in managing your network by templatizing your environment as a central source of truth. AWS provides a robust API that allows changes to be made quickly and to automate your environment’s processes. If you are doing a task 10 times, you can codify and automate it. This is the concept of Infrastructure as Code.</p>
<p><a href="https://aws.amazon.com/cloudformation/">AWS CloudFormation</a> is a service that provisions AWS resources quickly and simplifies your infrastructure management. A CloudFormation template describes the configuration of your resources, such as their property values. You can update templates to add, remove, or change the configuration of resources. This process keeps an accurate history of your resources, allowing you to track and document changes between different versions of your templates. So instead of recording changes in a ticketing system, you can reference changes in the versioning of CloudFormation templates.</p>
<p>CloudFormation provides a feature called <a href="https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/what-is-cfnstacksets.html">StackSets</a>, which extends the functionality of stacks by enabling you to create, update, or delete stacks across multiple accounts and Regions with a single operation. A stack set can be created in a central account, referred to as an administrator account, which provisions stacks and resource creation to target accounts.</p>
<p><a href="https://aws.amazon.com/organizations/">AWS Organizations</a> helps you centrally govern your accounts and scale your workloads on AWS. You can centrally manage billing, access control, compliance, security, and share resources across AWS accounts. Many existing AWS services take advantage of Organizations to streamline access control and management of resources.This post walks you through a solution that automates the association of transit gateway attachments to a transit gateway in a central account within your organization. It includes the following steps:</p>
<ul>
<li>Launching a template that creates a transit gateway in the central account.</li>
<li>Sharing the transit gateway to your organization with resource access manager.</li>
<li>Creation of IAM roles and permissions in a central account and child accounts.</li>
<li>Launching a stack set from a master account that automates the creation of a transit gateway attachment, which attaches to an existing <a href="https://aws.amazon.com/vpc/">Amazon VPC</a> based upon an AWS tag value.</li>
<li>Associating the transit gateway attachment with the transit gateway in the central account.</li>
<li>Creating a route table in the target account VPC with a configurable route CIDR address range and a destination to the transit gateway.</li>
<li>Implementing and deploying the transit gateway to populate routes within its own route table back to the target account VPC. (You can deploy this in your existing environment and across many target accounts in a single operation.)</li>
</ul>
<p>You can use the provided CloudFormation templates and code for a Lambda function to accomplish these tasks. The provided files can easily be modified to integrate in to your environment as part of your account creation and baselining.</p>
<h2>Prerequisites</h2>
<p>The following steps and services are instrumental in achieving this goal:</p>
<ul>
<li>Providing target accounts must have an existing VPC that is tagged for the VPC to be attached to the transit gateway attachment.</li>
<li>Providing target accounts with existing subnets with their VPC, as transit gateway attachment must have subnets from the attached VPC associated.</li>
<li>Transit Gateways are AWS Region specific. CloudFormation StackSets can be deployed across multiple regions. You will only be able to successfully deploy the stack set in the same AWS Region that the Transit Gateway resides with the provided templates. Additional changes can be made to the templates using CloudFormation mappings to make this a multi region solution.</li>
</ul>
<h2>Overview of AWS Organization architecture</h2>
<div id="attachment_2017" style="width: 852px" class="wp-caption alignnone"><img class="wp-image-2017 size-full" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2019/10/01/TransitGateway2.png" alt="Figure 1: Architectural diagram" width="842" height="787"><p class="wp-caption-text">Figure 1: Architectural diagram</p></div>
<p>Using a CloudFormation template, you can create a transit gateway in a central account and then share it with your organization using AWS <a href="https://docs.aws.amazon.com/ram/?id=docs_gateway">Resource Access Manager</a> (AWS RAM). You then can launch a stack set that automates the creation of a transit gateway attachment.</p>
<p>A Lambda function gathers metadata about a VPC based on a user-provided tag value. This metadata is relevant when creating and attaching the transit gateway attachment and VPC, along with route creation to the transit gateway based on a user-provided CIDR range.</p>
<p>The transit gateway in the central account is configured to auto-accept peering connections and to propagate its routing table to ensure bidirectional routing.</p>
<h2>Limitations</h2>
<ul>
<li>Overlapping CIDR ranges– Overlapping CIDR ranges are not allowed in the transit gateway routing table. Each VPC must have a unique CIDR.</li>
<li>Additional accounts–This solution works with multi-account environments. When deploying the stack set, you need an additional account to which to deploy.</li>
<li>Transit gateways – You can deploy five transit gateways per Region per account. You can modify the provided templates and Lambda function to send traffic to different transit gateways based on your standards.</li>
</ul>
<p>For more information about additional transit gateway limitations, see <a href="https://docs.aws.amazon.com/vpc/latest/tgw/transit-gateway-limits.html">Limits for Your Transit Gateways</a>.</p>
<h2>Granting IAM permissions for stack set operations</h2>
<p>Because stack sets perform stack operations across multiple accounts, you must have the necessary permissions defined in your AWS accounts before you can get started creating your first stack set.</p>
<p>In the central account, create an IAM role named AWSCloudFormationStackSetAdministrationRole. The role must have this exact name. You can do this by creating a stack from the following <a href="https://s3.amazonaws.com/cloudformation-stackset-sample-templates-us-east-1/AWSCloudFormationStackSetAdministrationRole.yml">CloudFormation template</a>. This role enables the following policy on your administrator account:</p>
<pre class=" language-json"><code class=" language-json"></code></pre>
<div class="prism-show-language"><div class="prism-show-language-label" data-language="Json">Json</div></div><pre class=" language-json" data-language="Json"><code class=" language-json"><span class="token punctuation">{</span>
	<span class="token property">"Version"</span><span class="token operator">:</span> <span class="token string">"2012-10-17"</span><span class="token punctuation">,</span>
	<span class="token property">"Statement"</span><span class="token operator">:</span> <span class="token punctuation">[</span><span class="token punctuation">{</span>
		<span class="token property">"Action"</span><span class="token operator">:</span> <span class="token punctuation">[</span>
			<span class="token string">"sts:AssumeRole"</span>
		<span class="token punctuation">]</span><span class="token punctuation">,</span>
		<span class="token property">"Resource"</span><span class="token operator">:</span> <span class="token punctuation">[</span>
			<span class="token string">"arn:aws:iam::*:role/AWSCloudFormationStackSetExecutionRole"</span>
		<span class="token punctuation">]</span><span class="token punctuation">,</span>
		<span class="token property">"Effect"</span><span class="token operator">:</span> <span class="token string">"Allow"</span>
	<span class="token punctuation">}</span><span class="token punctuation">]</span>
<span class="token punctuation">}</span></code></pre>
<p>The preceding template creates the following trust relationship:</p>
<div class="prism-show-language"><div class="prism-show-language-label" data-language="Json">Json</div></div><pre class=" language-json" data-language="Json"><code class=" language-json"><span class="token punctuation">{</span>
	<span class="token property">"Version"</span><span class="token operator">:</span> <span class="token string">"2012-10-17"</span><span class="token punctuation">,</span>
	<span class="token property">"Statement"</span><span class="token operator">:</span> <span class="token punctuation">[</span><span class="token punctuation">{</span>
		<span class="token property">"Effect"</span><span class="token operator">:</span> <span class="token string">"Allow"</span><span class="token punctuation">,</span>
		<span class="token property">"Principal"</span><span class="token operator">:</span> <span class="token punctuation">{</span>
			<span class="token property">"Service"</span><span class="token operator">:</span> <span class="token string">"cloudformation.amazonaws.com"</span>
		<span class="token punctuation">}</span><span class="token punctuation">,</span>
		<span class="token property">"Action"</span><span class="token operator">:</span> <span class="token string">"sts:AssumeRole"</span>
	<span class="token punctuation">}</span><span class="token punctuation">]</span>
<span class="token punctuation">}</span></code></pre>
<p>In each target account, create a service role named AWSCloudFormationStackSetExecutionRole that trusts the administrator account. The role must have this exact name. You can do this by creating a stack from the following <a href="https://s3.amazonaws.com/cloudformation-stackset-sample-templates-us-east-1/AWSCloudFormationStackSetExecutionRole.yml">CloudFormation template</a>. When you use this template, provide the name of the administrator account with which your target account must have a trust relationship.</p>
<p>Be aware that this template grants AdministratorAccess. After you use the template to create a target account execution role, you must scope the permissions in the policy statement to the types of resources that you are creating when using StackSets.</p>
<p>The target account service role requires permissions to perform any operations that your CloudFormation template specifies. Your target account always needs full CloudFormation permissions, which include permissions to create, update, delete, and describe stacks. The role created by this template enables the following policy on a target account:</p>
<div class="prism-show-language"><div class="prism-show-language-label" data-language="Json">Json</div></div><pre class=" language-json" data-language="Json"><code class=" language-json"><span class="token punctuation">{</span>
	<span class="token property">"Version"</span><span class="token operator">:</span> <span class="token string">"2012-10-17"</span><span class="token punctuation">,</span>
	<span class="token property">"Statement"</span><span class="token operator">:</span> <span class="token punctuation">[</span><span class="token punctuation">{</span>
		<span class="token property">"Effect"</span><span class="token operator">:</span> <span class="token string">"Allow"</span><span class="token punctuation">,</span>
		<span class="token property">"Action"</span><span class="token operator">:</span> <span class="token string">"*"</span><span class="token punctuation">,</span>
		<span class="token property">"Resource"</span><span class="token operator">:</span> <span class="token string">"*"</span>
	<span class="token punctuation">}</span><span class="token punctuation">]</span>
<span class="token punctuation">}</span></code></pre>
<p>The following trust relationship is created by the template. The administrator account’s ID shows as <em>central_account_id</em>:</p>
<div class="prism-show-language"><div class="prism-show-language-label" data-language="Json">Json</div></div><pre class=" language-json" data-language="Json"><code class=" language-json"><span class="token punctuation">{</span>
	<span class="token property">"Version"</span><span class="token operator">:</span> <span class="token string">"2012-10-17"</span><span class="token punctuation">,</span>
	<span class="token property">"Statement"</span><span class="token operator">:</span> <span class="token punctuation">[</span><span class="token punctuation">{</span>
		<span class="token property">"Effect"</span><span class="token operator">:</span> <span class="token string">"Allow"</span><span class="token punctuation">,</span>
		<span class="token property">"Principal"</span><span class="token operator">:</span> <span class="token punctuation">{</span>
			<span class="token property">"AWS"</span><span class="token operator">:</span> <span class="token string">"arn:aws:iam::central_account_id:root"</span>
		<span class="token punctuation">}</span><span class="token punctuation">,</span>
		<span class="token property">"Action"</span><span class="token operator">:</span> <span class="token string">"sts:AssumeRole"</span>
	<span class="token punctuation">}</span><span class="token punctuation">]</span>
<span class="token punctuation">}</span></code></pre>
<p>You can configure the trust relationship of an existing target account execution role to trust a specific role in the administrator account. If you delete the role in the administrator account and create a new one to replace it, you must configure your target account trust relationships with the new administrator account role, represented by central_account_id in the preceding example.</p>
<h2>Deploying the template and configuring a resource share</h2>
<p>In the central account in which to deploy the transit gateway, open the <strong>CloudFormation</strong> console.</p>
<p>On the <strong>Choose a template</strong> page, choose <strong>Upload a template file</strong> and select the provided transit-gateway.yaml file to upload. Choose <strong>Next</strong>.</p>
<p>On the <strong>Specify stack details</strong> page, provide a stack name along with the parameter values for the following fields:</p>
<ul>
<li><strong>Amazon-Side ASN</strong> – A private Autonomous System Number (ASN) for the Amazon side of a BGP session.</li>
<li><strong>Auto Accept Share Attachments</strong> – Indicate whether to accept cross-account attachment requests</li>
<li><strong>Auto Associate Route Table Association</strong> – Enable or disable automatic association with the default association route table.</li>
<li><strong>Automatic Route Propagation</strong> – Enable or disable automatic propagation of routes to the default propagation route table.</li>
<li><strong>DNS Support</strong> – Enable or disable DNS support.</li>
<li><strong>Equal Cost Multipath Protocol</strong> – Enable or disable equal-cost multi-path protocol (ECMP). You can use the ECMP to get higher VPN bandwidth by aggregating multiple VPN connections or Direct Connect VIFs. For this example, set the option to <strong>Disable</strong>, as you are not using VPN or Direct Connect VIF connections.</li>
</ul>
<p>At the time of publication, you cannot change or modify these parameters. To enable or disable these parameters, you must create a new transit gateway.</p>
<p>For this deployment, keep the default values. You can customize these values later for your own deployment.</p>
<p>The <strong>Configure stack options</strong> page doesn’t require you to enter any values or configuration changes. Choose <strong>Next</strong>.</p>
<p>On the <strong>Review</strong> page, verify your values and stack detail and choose <strong>Create stack</strong> to deploy the transit gateway.</p>
<p>The template takes a few minutes to deploy and reports CREATE_COMPLETE when finished.</p>
<p>In the VPC console, choose <strong>Transit Gateway</strong>. This page shows your deployed transit gateways and additional metadata. In the next step, you need the Transit Gateway ID.</p>
<p>When the transit gateway is available, you can share it across multiple accounts, an organization, and an organizational unit using AWS RAM. In this example, you share it across an organization. A significant benefit of sharing across an organization is that you can share resources without requiring the exchange of resource share invitations.</p>
<p>When using Organizations to share resources across an organization or organizational unit, you must enable sharing with your organization from your organization’s master account. Sharing within your organization can only be enabled by the master account. The following steps outline this process:</p>
<ul>
<li>Log in to the organization’s master account.</li>
<li>Open the Resource Access Manager</li>
<li>From the menu, choose <strong>Settings</strong>.</li>
<li>Choose <strong>Enable sharing within your AWS Organization</strong>.</li>
</ul>
<p>Log back in to the account that contains your transit gateway. In the Resource Access Manager console, choose <strong>Create resource share</strong>.</p>
<p>Provide a name to the resource share. <strong>For</strong> <strong>Select resource type</strong>, choose <strong>Transit Gateways</strong>. This step populates a field where you can select your transit gateway.</p>
<p>Under <strong>Principals</strong>, you can add the accounts, organizational units, or your organization with which to share. This step allows you to limit what accounts can view the transit gateway as an available resource to which to attach. For this example, share with the entire organization.</p>
<p>Choose <strong>Create resource share</strong>. Now you have a transit gateway shared with your organization. Because you shared across the organization, you don’t have to accept sharing invitations in the child accounts.</p>
<h2>Deploying the VPC with a template in the central account as a stack set</h2>
<p>In the central account, create and deploy a stack set. This stack set deploys CloudFormation stacks in target accounts that create a Lambda function to perform a lookup for your VPC and its associated subnets and default route table, based upon a user-provided tag value.</p>
<p>You can use that information to create and attach the transit gateway attachment and VPC to each other. The transit gateway attachment becomes associated with the central account transit gateway. The target VPC’s default route table populates with a configurable route with the destination to the central account transit gateway. The transit gateway’s route table then populates routes back to the target VPC.</p>
<p>This example deploys a Lambda function in each target account as part of the automation of transit gateway attachments. The code is in a .zip file named lambda.zip. Upload the file to an S3 bucket that allows access permissions to the child account. <a href="https://aws.amazon.com/s3/">Amazon S3</a> bucket policies support the PrincipalOrg, which allows any account that originates from the same Organizations ID to access resources from with S3.</p>
<p>The following is a sample S3 bucket policy to allow the GetObject and ListBucket actions within your organization. Replace S3 bucket ARN and o-id with your own resource values:<code></code></p>
<div class="prism-show-language"><div class="prism-show-language-label" data-language="Json">Json</div></div><pre class=" language-json" data-language="Json"><code class=" language-json"><span class="token punctuation">{</span>
	<span class="token property">"Version"</span><span class="token operator">:</span> <span class="token string">"2012-10-17"</span><span class="token punctuation">,</span>
	<span class="token property">"Statement"</span><span class="token operator">:</span> <span class="token punctuation">[</span>
		<span class="token punctuation">{</span>
			<span class="token property">"Sid"</span><span class="token operator">:</span> <span class="token string">"S3AccessToTgwLambdaCode"</span><span class="token punctuation">,</span>
			<span class="token property">"Action"</span><span class="token operator">:</span> <span class="token punctuation">[</span>
				<span class="token string">"s3:GetObject"</span><span class="token punctuation">,</span>
				<span class="token string">"s3:ListBucket"</span>
			<span class="token punctuation">]</span><span class="token punctuation">,</span>
			<span class="token property">"Effect"</span><span class="token operator">:</span> <span class="token string">"Allow"</span><span class="token punctuation">,</span>
			<span class="token property">"Resource"</span><span class="token operator">:</span> <span class="token punctuation">[</span>
				<span class="token string">"S3 bucket ARN"</span><span class="token punctuation">,</span>
				<span class="token string">"S3 bucket ARN/*"</span>
			<span class="token punctuation">]</span><span class="token punctuation">,</span>
			<span class="token property">"Condition"</span><span class="token operator">:</span> <span class="token punctuation">{</span>
				<span class="token property">"StringEquals"</span><span class="token operator">:</span> <span class="token punctuation">{</span>
					<span class="token property">"aws:PrincipalOrgID"</span><span class="token operator">:</span> <span class="token string">"o-id"</span>
				<span class="token punctuation">}</span>
			<span class="token punctuation">}</span><span class="token punctuation">,</span>
			<span class="token property">"Principal"</span><span class="token operator">:</span> <span class="token string">"*"</span>
		<span class="token punctuation">}</span>
	<span class="token punctuation">]</span>
<span class="token punctuation">}</span></code></pre>
<p>You can modify this sample policy to allow external account access, but this example locks down the policy to your organization.</p>
<p>In the <strong>CloudFormation</strong> console, choose <strong>StackSets</strong>, <strong>Create StackSet</strong>.</p>
<p>On the <strong>Choose a template</strong> page, choose <strong>Upload a template file</strong>. Select the provided transit-gateway-association.yaml file and choose <strong>Next</strong>.</p>
<p>On the <strong>Specify stack details</strong> page, provide a name for the stack set along with the parameter values for the following fields, then choose <strong>Next</strong>.</p>
<ul>
<li><strong>VPC Tag Value</strong> – Tag values of VPCs in your child accounts that you would like to associate with the transit gateway (comma-separated). Only the tag value is required. For example, if your tag is Name:Production, you only have to provide the value Production.</li>
<li><strong>Transit Gateway Id</strong> – ID of the central account transit gateway.</li>
<li><strong>Route Destination CIDR</strong> – Destination route for traffic to the central account transit gateway.</li>
<li><strong>S3 Bucket</strong> – S3 bucket for transit gateway attachment Lambda code.</li>
<li><strong>S3 Key</strong> – The key location of the Lambda function .zip for transit gateway attachment.</li>
</ul>
<p>On the <strong>Configure StackSet options</strong> page, enter AWSCloudFormationStackSetExecutionRole for the IAM execution role name. This field should already have this value populated. You do not need an optional IAM admin role ARN. Choose <strong>Next</strong>.</p>
<p>In the <strong>Set deployment options</strong> screen, provide the account numbers or the organizational unit ID in which to deploy the solution. You can use either value depending on your use case. The benefit of selecting accounts is that you limit the deployment to the specific accounts in which to deploy. The organizational units deploy in all accounts within that organizational unit.</p>
<p>You can specify the Region or Regions that the solution deploys in for each account. You can also set deployment options to deploy one at a time, or increase the options to deploy in multiple accounts at the same time. However, transit gateways are Region-specific, and you must deploy the stack set in the same Region in which the transit gateway resides. You can further customize the template with multiple transit gateways in multiple Regions, which could have mappings defined to associate the correct transit gateway with the deployment Region.</p>
<p>After you enter the preceding information, choose <strong>Next</strong>.</p>
<p>On the following screen, verify your settings and choose <strong>Submit</strong>. This step begins the operation of deploying the stack set to the target account.</p>
<p>You can add additional accounts as stack instances by navigating to the console for the newly created stack set. Choose <strong>Actions</strong>, <strong>Add new stacks to </strong>StackSet, and repeat the preceding process.</p>
<h2>Monitoring activity</h2>
<p>After the stack set is running, you can log in to the target account. In the CloudFormation console, view the newly created stack. The <strong>Resources</strong> tab shows a list of created items.</p>
<p>To verify success, log in to a target account. In the VPC console, in the left navigation pane, choose <strong>Transit Gateway Attachments</strong>. You should see a transit gateway attachment associated with the transit gateway in the central account. You can navigate to your route tables and verify that a route is now in the VPC’s default route table with the transit gateway as the destination.</p>
<p>In the central account, in the VPC console, choose <strong>Transit Gateway Routes</strong>. There you can verify that there are routes back to the target VPCs.<strong>&nbsp;</strong></p>
<h2>Conclusion</h2>
<p>The provided templates serve as a starting point to build out more complex solutions that you can integrate into your environment. You can modify the stack set to handle multiple transit gateways that could reside in different Regions or for different workloads.</p>
<p>You can also build logic into the CloudFormation template to handle conditions and determine which transit gateways to attach to or how to construct the route tables. This process can become part of the foundation for automating your environment and network management across multiple VPCs and AWS accounts.</p>


## License

This library is licensed under the MIT-0 License. See the LICENSE file.

