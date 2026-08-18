# Set Up SAP Build Work Zone

For this mission, SAP Build Work Zone is optional.
The Joule conversational search capability does not require integration with SAP Build Work Zone, standard edition. 

For other Joule capabilities, SAP Build Work Zone is required.

You can only connect to one Work Zone instance per Joule Instance. For more information, see [SAP Help Portal](https://help.sap.com/docs/joule/integrating-joule-with-sap/prerequisites?version=LATEST&locale=en-US)

In this mission, you use Build Work Zone, standard edition. For detailed information about setting up SAP Build Work Zone, see [SAP Help Portal](https://help.sap.com/docs/build-work-zone-standard-edition).
   
For supported Joule regions, see [Data Centers Supported by Joule](https://help.sap.com/docs/joule/serviceguide/data-centers-supported-by-joule).



### Set Up Build Work Zone

1. In your subaccount, navigate to Security --> Trust Configuration.
    Establish Trust in your Identity Authentication Tenant.

    <img src="images/jouledev_111_trust.png" alt="Establish Trust" width="600">

2. In your subaccount, navigate to Services --> Instances and Subscriptions.

3. Select "Create".

4. Choose "SAP Build Work Zone, standard edition" (or your edition).

5. As a service plan, select Subscription "standard" (or your plan).

    <img src="images/workzone_101_create.png" alt="select Service and plan" width="400">

6. Select Create. Work Zone will be subscribed.

    <img src="images/workzone_102_subscribed.png" alt="Build Work Zone subscribed" width="500">

7. Navigate to "Security" --> "Users". Select your business user, which authenticates against your Identity Authentication Tenant. Assign the role collection "Launchpad_Admin" if it is not already assigned.

    <img src="images/workzone_103_addrole.png" alt="Build Work Zone subscribed" width="600">

8. Start the Build Work Zone Application in "Instances and Subscriptions". Click on the Application name.

9. You should see the Site Directory.

    <img src="images/workzone_104_sitedirectory.png" alt="Build Work Zone Site Directory" width="600">

Congrats, you can continue with the Joule setup.

---
