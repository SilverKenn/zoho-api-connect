# Zoho API Integration

Pre-defined class for connecting to Zoho API, includes oauth2 related functionality.

## Installation
Just upload the plugin zip file on Wordpress Admin -> Plugins -> Add New -> Upload Plugin

## Usage 
The pre-defined methods are already authenthecated, 

### Pushing data to Zoho using send_leads method
````php
zh_API::send_leads( $form_data, $url, $request_type );
````
### Parameters

$form_data - a json format of data that will be push to zoho, refer to <a href="https://www.zoho.com/crm/developer/docs/api/insert-records.html" target="_blank">Zoho Insert Record API</a> for better explanation of data construction (default value: none, required )

$url - The URL where you want the data to be push (defaul value: https://www.zohoapis.com/crm/v2/Leads)

$request_type - Type of request, can be POST or PUT ( default value: POST)

### example usage

````php
// Construct and mapped the fields on your Zoho Forms and assign the value from your form data
$data = [
	'First_Name' => 'John',
	'Last_Name' => 'Doe',
	'Email' =>  'john@doe.com',
	'Phone' => '1234567890',
];


// We need to format and construct the array based on Zoho API documentation
$lead = [
	'data' => [ $data ], 
	'trigger' => ['approval', 'workflow', 'blueprint'],
];

// convert the array to JSON then call zh_API::send_leads to push the data on Zoho
$send = zh_API::send_leads( json_encode($lead) );
````


### Logging
This method is helpful for debugging and checking data. You can capture the response or even the data the this being sent via curl then add it on a log file for data analyzing
````php
zh_API::log( $file_name, $content,  $prepend_data, $use_exact_filename, $directory );
````
### Parameters
$file_name - File name for the log, (default value: none, required) 

$content - Log Content (default value: none, required) 

$prepend_date - remove exact date and time for each log ( default value: false ) 

$use_exact_filename - use $file_name as the exact file name ( default value: false ) 

$directory - directory the log will be stored ( default value: '/logs' )

### example usage

In the above example, we stored the request to $send variable, we can simply save the variable content into a log file 

````php
zh_API::log( 'data.json', json_encode($d), 0, 1 );
````
This will create a file or append the $send content to a file wp-cotentnt/plugins/zoho-api/logs/data.json


### eform data connection  
This method was based on your current form set-up and fields, For additional informations about eform hooks, filters, data or database structure, you can refer to their docs
https://wpquark.com/kb/fsqm/fsqm-api/fsqm-pro-developers-handbook/

Due to data structure complexity, I ended up creating two functions to pull the specific value from the data being provided by eforms 

````php
//get and pull array key value form an array to a different array with connected keys
function _mpp( $d, $type, $id, $options = 'label' ) {
	if ( isset( $d[$type][$id] ) && isset( $d['data']->$type[$id] ) && isset( $d['data']->$type[$id]['options'][0] ) && $d['data']->$type[$id]['options'][0] !== '' )
		return $d[$type][$id]['settings']['options'][ $d['data']->$type[$id]['options'][0] ][$options];
}
````
````php
// Pull array key value from specific object
function _gmp( $keys, $id = 0, $key = 'value' ) {
	if ( isset( $keys[$id][$key] ) ) return $keys[$id][$key];
}
````

Actual Hook for Zoho Data mapping
````php
// lets hook our function on ipt_fsqm_hook_save_insert more info here https://wpquark.com/kb/fsqm/fsqm-api/apis-on-form-submission-handling/
add_action( 'ipt_fsqm_hook_save_insert', '_foward_captured_data_to_zoho' );

// Function Construction 
function _foward_captured_data_to_zoho( $submitted ) {

	// well check the form ID on where to execute this hook, stop code execution if the form id is not 54 
	if ( $submitted->data->form_id != "54" ) return;

	// lets re-construct the $submitted variable being pass by eforms
	$d = [
		'data' => $submitted->data,
		'mcq'  => $submitted->mcq,
		'pinfo'  => $submitted->pinfo
	];

	// Constructing and mapping the data to Zoho field
	$data = [
		'Owner' => '4110100000000225013',
		'Layout' => '4110100000000242038',
		'Where_are_you_applying_for_a_visa_to_Russia' => _mpp($d, 'mcq', 1),
		'Which_country_issued_your_passport'	=> _mpp($d, 'mcq', 3 ),
		'Visa_Service'	=> _mpp($d, 'mcq', 4),
		'Invitation_Letter_Needed' => _mpp($d, 'pinfo', 0),
		'Visa_Invitation' => _mpp($d, 'pinfo', 0 ) == 'Business' ? _mpp($d, 'mcq', 6) : _mpp($d, 'mcq', 5),
		'Number_of_Visa_Applicants' => _gmp( $d['data']->mcq, '8' ),
		'Processing_Time' => _mpp($d, 'mcq', 9),
		'First_Name' => _gmp( $d['data']->pinfo, '76'),
		'Last_Name' => _gmp( $d['data']->pinfo, '77'),
		'Email' =>  _gmp( $d['data']->pinfo, '2'),
		'Phone' => _gmp( $d['data']->pinfo, '3'),
		'Recipient' =>_gmp( $d['data']->pinfo, '4', 'recipient' ),
		'Recipient_Address' => _gmp( $d['data']->pinfo, '4', 'line_one' ),
		'City_State_Zip_Code' => _gmp( $d['data']->pinfo, '4', 'line_two' ),
		'Date_of_Trip' => _gmp( $d['data']->pinfo, '6'),
		'Date_of_Departure_from_Russia' => _gmp( $d['data']->pinfo, '8'),
		'Cities_to_Visit' => _gmp( $d['data']->pinfo, '9'),
		'Date_of_Arrival_in_Russia' => _gmp( $d['data']->pinfo, '7'),
		'Name_Address_of_Accommodation' => _gmp( $d['data']->pinfo, '11'),
		'FedEx_Shipping_Label_14' => _gmp( $d['data']->mcq, '17' ),
		'Next_Day_Return_Delivery_59' => _gmp( $d['data']->mcq, '14'),
		'FedEx_pickup_8' => _gmp( $d['data']->mcq, '19' ),
		'Enter_Discount_Coupon' => _gmp( $d['data']->pinfo, '12', 'coupon' ),
		'Payment_Method' => _gmp( $d['data']->pinfo, '12', 'pmethod' )
	];


	// Additional field for Sub-form Mapping
	$applicants = $d['data']->pinfo[75]['values'];
	$visa_applicants  = [];
	if ( $applicants ) {
		foreach ($applicants as $app ) {
			$visa_applicants[] = [
				'First_Middle_names_as_in_passport' => $app[0],
				'Last_Name_as_in_passport' => $app[2],
				'Gender' => $app[3],
				'Date_of_Birth' => $app[4],
				'Passport_Num' => $app[5],
				'Place_of_Birth' => $app[8],
				'Date_of_Issue' => $app[6],
				'Date_of_Expiration' => $app[7]
			];
		}
	}
	if ( $visa_applicants ) {
		$data['Visa_Applicant'] =  $visa_applicants;
	}
	
	// format the data for Zoho records API requirements
	$lead = [
		'data' => [ $data ], 
		'trigger' => ['approval', 'workflow', 'blueprint'],
	];
	
	// Sending the data to Zoho 
	$send = zh_API::send_leads( json_encode($lead) );
	
	// check for any file attachments
	$files = _gmp( $d['data']->freetype, '1', 'id' );
	
	// check if $send request is successfull and if there's attach files on the form
	if( isset( $send->data[0]->status) && $send->data[0]->status == 'success' && $files ) {

		// Get the files from databases
		$atts = zh_API::files($files);
		$id = $send->data[0]->details->id;
		$upload = [];
		if ( $atts ) {

			// Loop through each file 
			foreach ($atts as $att ) {

				// Upload file to Zoho
				$upload[] = zh_API::upload( $att, $id);
			}

			// Store the upload request response to a file
			zh_API::log('upload-response.json', json_encode($upload), 0, 1);
		}
	}
	
	//For debugging and verification
	//We'll create logs and store multiple data as its the only way we can check the response and request data

	// Store the re-constructed data form $submitted variable to data.json
	zh_API::log('data.json', json_encode($d), 0, 1);

	// Store the send_lends response from zoho to send-response.json
	zh_API::log('send-response.json', json_encode($send), 0, 1);

	// Store our send_lends request data and save it to send-response.json
	zh_API::log('send-response.json', json_encode($data), 0, 1);
	
}
````
