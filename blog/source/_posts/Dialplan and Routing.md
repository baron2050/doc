---
title: Dialplan and Routing
---

We will introduce four new modules (drouting, dialplan, permissions, and
group) that will help you to route and secure calls from and to PSTN.

## The dialplan module

The dialplan module provides a single function named dp_translate(). This
function translates any variables according to a list of regular expressions. It is
very useful to normalize dialed numbers and caller IDs and select different types
of routing services.

![image-20210419091444272](../img/Dialplan and Routing/image-20210419091444272.png)

## PSTN routing Receiving calls from PSTN

You can do this in a static way by hardcoding the IP addresses directly in your
script:
{
..
if ( ($si=="10.10.10.5" && $sp==5060 && $proto=="UDP") ||
($si=="10.10.10.9" && $sp==5060 && $proto=="UDP") ) {

#trusted GW, skip digest authentication

} else if (!proxy_authorize("", "subscriber")) {
proxy_challenge("", "0");
exit;
}
..
}

### The permissions module

The equivalent of the preceding script, but using the permissions module, is as
follows:
loadmodule "permissions.so"
modparam("permissions","db_url",
"mysql://user:pwd@localhost/opensips")
{
..
if ( check_source_address("1") ) {

#trusted GW

} else if (!proxy_authorize("", "subscriber")) {
proxy_challenge("", "0");
exit;
}
..
}

The permissions module provides you with a more flexible function named
check_address(group, IP, port, proto) that can receive the IP, port, and
protocol as parameters rather than automatically using the network source
information

###  dialplan module

{
..
if ( check_source_address("1") ) {

#trusted GW, normalize to E.164

if ( $rU=~"^00" ) { # international 00 format
strip(2);
} else if ( $rU=~"^\+" ) { # international + format
strip(1);
} else if ( $rU=~"^0" ) { # national format
strip(1);
prefix("44"); # add the UK international prefix
}
}
..
}

You can also use a dialplan set for this:
loadmodule "dialplan.so"
modparam("dialplan","db_url",
"mysql://user:pwd@localhost/opensips")
{
..
if ( check_source_address("1") ) {

#trusted GW, normalize to E.164

if ( !dp_translate("10","$rU/$rU") ) {
send_reply("404","Bad PSTN number");
exit;
}
}
..
}
The corresponding DB content is as follows:
Set ID Match operator Match exp Subst exp Replace exp
![image-20210419092746686](../img/Dialplan and Routing/image-20210419092746686.png)



### alias_db

loadmodule "alias_db.so"
modparam("alias_db","db_url",
"mysql://user:pwd@localhost/opensips")
{
..
if ( check_source_address("1") ) {

trusted GW, normalize to E.164

if ( !dp_translate("10","$rU/$rU") ) {
send_reply("404","Bad PSTN number");
exit;
}

use the "dids" aliases table and do not

use the domain part when looking up for alias

if ( alias_db_lookup("dids","d") ) {
send_reply("404","DID not found");
exit;
}
}
..
}

## PSTN routing  Sending calls to PSTN

The handling of the calls towards the PSTN side can be split into the following
steps:
1. Identifying and normalizing the dialed number.
2. Authorizing the PSTN gateway usage.
3. Presenting the proper caller ID for the PSTN side.
4. Routing the appropriate PSTN gateway.

The identification can be hardcoded in the script. In this example, we have a
setup where eight-digit numbers are used as subscriber IDs and five-digit
numbers starting with number 4 are service numbers:
if ( !($rU=~"^[1-9][0-9]{7}$" || $rU=~"^4[0-9]{4}$") ) {

#forward to the PSTN GW

}

if ( !dp_translate("2","$rU") ) {

#forward to the PSTN GW

}
The following is the corresponding DB content:

![image-20210419093759022](../img/Dialplan and Routing/image-20210419093759022.png)

## Script samples

The script for the dynamic routing module is as follows:
Gateway type 1 for inbound gateways (DID providers sending calls to us)
Gateway type 2 for outbound gateways (PSTN carriers)
Rules group 10 for DID matching; the attribute will hold the SIP username
behind this DID
Rules group 20 for prefix-based routing to gateways

The following is the dr_gateways table:

![image-20210419094615118](../img/Dialplan and Routing/image-20210419094615118.png)

The dr_rules table is as follows:

![image-20210419094650585](../img/Dialplan and Routing/image-20210419094650585.png)

The script for the dialplan module is as follows:

Group 1 to normalize the in and out PSTN numbers in the E.164 format

![image-20210419094745348](../img/Dialplan and Routing/image-20210419094745348.png)



```c
# Here we have the scripted PSTN inbound sections:
# we have here only initial request !
# handle inbound traffic – first check if it comes
# from our DID providers (use IP auth with DRouting)
if ( is_from_gw("1") ) {
# trusted inbound GW, normalize to E.164
if ( !dp_translate("1","$rU/$rU") ) {
send_reply("404","Bad PSTN number");
exit;
}
# use DR to match DIDs- full number matching, no routing
if ( !do_routing("10","LC",,"$avp(username)") ) {
send_reply("404","DID not found");
exit;
}
# set the new URI based on DID translation
$rU = $avp(username);
} else if (!proxy_authorize("", "subscriber")) {
proxy_challenge("", "0");
exit;
}
These are the scripted PSTN outbound sections:
....
# we have here only initial request !
# here we decide how to route further the call
# is it a call for PSTN ?
if ( dp_translate("1","$rU/$rU") ) {
# recognized and normalized to E.164
# use DR to route with weights
if ( !do_routing("20","W") ) {
send_reply("503","No route found");
exit;
}
xlog ("first gateway is $ru \n");
# arm failure route to catch bad GWs
t_on_failure("dr");
t_relay();
exit;
}
.....
failure_route[dr] {
if (t_was_cancelled()) {
exit;
}
# a gateway failure is only for 5xx and 6xx replies
# or for a local timeout (no reply was received)
if ( !(t_check_status("[56][0-9][0-9]") || (t_check_status("408")
&& t_local_replied("all")) ) )
exit;
if (use_next_gw()) {
xlog ("next gateway is $ru \n");
t_on_failure("dr");
t_relay();
exit;
} else {
t_reply("503", "Service not available");
exit;
}
}
```

# dialog

You can set a dialog variable after creating a dialog in the initial request. You can only read a dialog variable after a dialog matching command such as loose_route() or match_dialog()

To attach a value to a dialog, you can use the store_dlg_value(name,val)
function or directly set the variable using $dlg_val(name). See the following
examples:
store_dlg_value("type","residential");

#or

$dlg_val(type) = "residential";
To recover and/or set a dialog variable, you can again use the
fetch_dlg_value(name,pvar) function or directly set the variable using
$dlg_val(name). See the following examples:
fetch_dlg_value("type","$avp(dlgtype)");

#or

$avp(dlgtype) = $dlg_val(type);

