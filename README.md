# public

```python

def can_be_processed(client_phonenumber=None):
	phonenumber = "+"+client_phonenumber
	#Check if customer exist
	get_customer = None
	try:
		get_customer = Clients.objects.get(telephone = phonenumber)
	except:
		pass

	if not get_customer:
		return {"status": False, "message":"Customer not found", 'status_code':"CUSTOMER_NOT_FOUND"}

	# Check if API is accessible
	api_access_values = steamaco_api_values()
	headers = {
		'Authorization': api_access_values["Authorization"],
	}
	response = requests.get(api_access_values['host'], params='', headers=headers)
	status_code = int(response.status_code)
	if status_code == 200:
		return {"status": True, "message":"", 'status_code':""}
	else:
		return {"status": False, "message":"API not accessible", 'status_code':"API_NOT_ACCESSIBLE"}


@api_view(['PUT', 'GET', 'POST'])
def smswebhook(request):
	data = request.data
	params = request.query_params

	### Validate phone number ####
	try:
		phone_number = str(data['from']).replace('+','').replace(' ','')
	except:
		return Response({"message":"Invalid phone number"}, status=status.HTTP_400_BAD_REQUEST)

	try:
		python_phone_number = phonenumbers.parse("+"+phone_number, None)
	except:
		return Response({"message":"Incorrect phone number format. Ensure number includes country code"}, status=status.HTTP_400_BAD_REQUEST)

	if not phonenumbers.is_valid_number(python_phone_number):
		return Response({"message":"Invalid phone number"}, status=status.HTTP_400_BAD_REQUEST)

	mobile_carrier = str(carrier.name_for_number(python_phone_number, "en"))
	country = geocoder.country_name_for_number(python_phone_number, "en").lower()
	try:
		mobile_carrier_short = mobile_carrier.split()[0].lower()
	except:
		return Response({"message":"Incorrect phone number format. Ensure number includes country code"}, status=status.HTTP_400_BAD_REQUEST)

	is_mtn = False
	is_orange = False
	ussd_code = ''
	if mobile_carrier_short == 'mtn' and country == 'cameroon':
		is_mtn = True
		is_orange = False
		ussd_code = '*126#'
	elif mobile_carrier_short == 'orange' and country == 'cameroon':
		is_mtn = False
		is_orange = True
		ussd_code = '#150*50#'
		return Response({"message":"Unsupported Carrier"}, status=status.HTTP_400_BAD_REQUEST)
	else:
		return Response({"message":"Unsupported Carrier"}, status=status.HTTP_400_BAD_REQUEST)
	###

	text = data["text"].strip().replace(",","")

	content = ''.join(text.split())
	command = content[0].upper()
	if command == 'T':
		#Topup
		try:
			amount = int(content[1:])
		except:
			return Response({"message":"Invalid amount"}, status=status.HTTP_400_BAD_REQUEST)

		get_user = system_user

		process_or_not = can_be_processed(client_phonenumber=phone_number)
		print("process_or_not: ", process_or_not)
		if process_or_not['status'] == True:
			internal_reference = get_new_uuid()
			description = "SMS Top up"
			external_reference = "metertopup"
			system_transaction = initiate_transaction(user = get_user, do_momo=True, is_mtn = is_mtn, is_orange = is_orange, is_collection=True, is_disbursement=False, phone_number=phone_number, amount=amount, description=description, internal_reference=internal_reference, external_reference=external_reference, notify_url=None)

			content = {"reic_reference": system_transaction.code, "operator": system_transaction.operator, "ussd_code":ussd_code}
			return Response(content, status=status.HTTP_200_OK)

		else:
			return Response({"message":process_or_not['message']}, status=status.HTTP_400_BAD_REQUEST)

	else:
		return Response({"message":"Invalid Command"}, status=status.HTTP_400_BAD_REQUEST)



def topup_meter(get_system_transaction):

	phonenumber = "+"+get_system_transaction.phone_number
	print("phonenumber: ", phonenumber)

	get_customer = None
	try:
		get_customer = Customers.objects.get(telephone = phonenumber)
	except:
		pass
	#

	if get_customer:
		print("CUSTOMER_FOUND")
		customer_id = get_customer.customer_id
		amount = str(get_system_transaction.amount)
		reference = get_system_transaction.code

		api_access_values = steamaco_api_values()

		headers = {
			'Authorization': api_access_values["Authorization"],
		}

		payload = {"amount": amount, "category": "PAY", "reference": reference}		
		response = requests.post(api_access_values['host']+'/customers/'+customer_id+'/transactions/', data=payload, headers=headers)
		
		has_errors = False
		try:
			response_json = response.json()
		except:
			has_errors = True

		if not has_errors:
			pass

		return True

	else:
		print("CUSTOMER_NOT_FOUND")
		return "CUSTOMER_NOT_FOUND"


```
