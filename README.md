# ZohoCRM-Convert-Record-Across-Modules
A Deluge script that replicates the conversion of a record across different modules in Zoho CRM by copying info to a new record in a different module and deleting the old record.

## Core Idea
Zoho CRM has a default conversion button that lets users convert Leads to Contacts on a click of a button. This functionality however, is limited to only the 2 aforementioned modules. This script aims to replicate the conversion button should you need to convert records across modules that aren't Leads to Contacts. This is done by copying all the necessary fields and related lists from the original module to the new, then deleting the old record.

## Example Case
You are running an insurance business with two different types of client accounts, "Households" and "Businesses". You manage these 2 different types of accounts in CRM by having 2 modules with the aforementioned types. At any point of time, your staff needs to convert "Household" records to "Businesses" and vice versa.

## Configuration
* The two inter-related conversion modules should be set up with the relevant fields mapped.
* The script should be written in a custom button for each module so users can easily execute the conversion from the CRM record page.
* Zoho CRM Connection needed:
  * ZohoCRM.modules.ALL

## Tutorial
### Get the Old Record Fields and Create a New Record
Get all necessary fields from the original (old) record, then create a new record in the destination module. Get the record ID from the `createRecord` function for updates later on in the script.

```javascript
//Change below to the relevant old & new modules
oldModule = "Households";
newModule = "Businesses";

oldRecord = zoho.crm.getRecordById(oldModule,recordId);

//Get all Necessary Fields - add/modify as needed
recordName = oldRecord.get("Name");
email = oldRecord.get("Primary_Email");

//Create Record
rMap = Map();
rMap.put("Name",recordName);
rMap.put("Primary_Email",email);
newRecord = zoho.crm.createRecord(newModule, rMap);
newRecordId = newRecord.get("id");
info newRecordId;
```
### Get all necessary Related Lists and Modify it to the New Record
To truly convert a record, we need to account for all **related lists**. The common related lists in records are Contacts, Deals, Notes and Activities (Tasks, Events, Calls). This list may differ/be longer on a case-to-case basis depending on the nature of your modules, but for the purpose of this demonstration, we will focus on the four mentioend.

#### Related Contacts
In this example, "Household" and "Business" are both lookup fields on the Contact module. You will need to change this based on how the your modules are set up in relation to Contacts. A `for loop` is used to iterate through every Contact (if there are more than one).
```javascript
allContacts = zoho.crm.getRelatedRecords("Related_Contacts", oldModule, recordId); //Change this to Contacts
for each c in allContacts
{
	delete = zoho.crm.updateRecord("Contacts", c.get("id"), {"Household" : ""});
	update = zoho.crm.updateRecord("Contacts", c.get("id"), {"Business" : newRecordId});
	info delete;
	info update;
}
```

#### Related Deals
Similar to above, you will need to change this based on how the your modules are set up in relation to Deals. A `for loop` is used to iterate through every Deal (if there are more than one).
```javascript
allDeals = zoho.crm.getRelatedRecords("Deals", oldModule, recordId);
if (allDeals.size() > 0)
{
  for each d in allDeals
  {
	  delete = zoho.crm.updateRecord("Deals", d.get("id"), {"Household_Name" : ""});
	  update = zoho.crm.updateRecord("Deals", d.get("id"), {"Business_Name" : newRecordId});
	  info delete;
	  info update;
  }
}
```

#### Related Notes
The Notes related list has a universal structure so changes to the script are not needed. Here, we need to copy the **Note Title** and **Note Content** from the old record to the new. A `for loop` is used to iterate through every note. It will not be necessary to delete the note(s) in the old record as they will be deleted later along with the record deletion.

```javascript
allNotes = zoho.crm.getRelatedRecords("Notes", oldModule, recordId);
if (allNotes.size() > 0)
{
  for each n in allNotes
  {
	  notesMap = {"Parent_Id":newRecordId,"Note_Title":n.get("Note_Title"),"Note_Content":n.get("Note_Content"),"$se_module": newModule};
	  create = zoho.crm.createRecord("Notes",notesMap);
	  info create;
}
```

#### Open Activities
Get all Activities on the old record, iterate through a `for loop`, then modify the Activity by linking it to the new record. 
* `"$se_module"` determines the **module** the activity should lookup to
* `What_Id` links the activity to the specific **record** lookup
* Since there are 3 different types of activities (Tasks, Events, Calls), `a.get("Activity_Type")` is used as an identifier for the system in `updateRecord`.

```javascript
allAct = zoho.crm.getRelatedRecords("Activities", oldModule, recordId);
if (allAct.size() > 0)
{
for each a in allAct
	{
		idMap = Map();
		idMap.put("id",newRecordId);
		aMap = Map();
		aMap.put("$se_module",newModule);
		aMap.put("What_Id",idMap);
		updateAct = zoho.crm.updateRecord(a.get("Activity_Type"), a.get("id"), aMap);
		info updateAct;
	}
}
```

#### Closed Activities
When activites related to records are closed, they are moved to a different related list in the system called "Activities_History". For traceability, it's a good practice that closed activites are also migrated to the new record.

```javascript
allPastAct = zoho.crm.getRelatedRecords("Activities_History", oldModule, recordId);
if (allPastAct.size() > 0)
{
	for each ap in allPastAct
	{
		idMap = Map();
		idMap.put("id",newRecordId);
		aMap = Map();
		aMap.put("$se_module",newModule);
		aMap.put("What_Id",idMap);
		updatePastAct = zoho.crm.updateRecord(ap.get("Activity_Type"), ap.get("id"), aMap);
		info updatePastAct;
	}
}
```

### Download & Upload Attachments
To leave no stones unturned, all attachment(s) on the old record will also be migrated by downloading the attachment(s), and uploading to the new record.
```javascript
//Download & Upload Attachments
attachment = invokeurl
[
	url: "https://www.zohoapis.com/crm/v2/"+oldModule+"/"+recordId+"/Attachments"
	type: GET
	connection: "zohocrm" //Change this to your connection name
];
if (attachment.get("data").size() > 0)
{
	for each atm in attachment.get("data")
	{
		download = invokeurl
		[
			url: "https://www.zohoapis.com/crm/v2/"+oldModule+"/"+recordId+"/Attachments/"+atm.get("id")
			type: GET
			connection: "zohocrm" //Change this to your connection name
		];
		
		upload = zoho.crm.attachFile(newModule,newRecordId,download);
		info upload;
	}
}
```

### Delete Old Record
When the transition is completed, delete the old record from the original module.
```javascript
delete = invokeurl
[
	url: "https://www.zohoapis.com/crm/v2/"+oldModule+"?ids="+recordId+"&wf_trigger=true"
	type: DELETE
	connection: "zohocrm" //Change this to your connection name
];
info delete;
```