..	1
 This work is licensed under a Creative Commons Attribution 3.0 Unported	2
 License.	3
4
 http://creativecommons.org/licenses/by/3.0/legalcode 	5
6
==========================================	7
Replace home grown WSGI layer with Pecan	8
==========================================	9
10
https://blueprints.launchpad.net/neutron/+spec/pecan-switch 	11
12
This document describes a plan to replace the current home-grown WSGI	13
framework, including REST controllers, with a solution entirely based	14
on the Pecan framework [1]_	15
16
The specification discussed in this document assumes that the REST controllers	17
will dispatch calls to the plugin in a different way, leveraging a new	18
interface which is thoroughly discussed in [2]_	19
20
Problem Description	21
===================	22
23
This specification addresses a number of issues arising from the fact that	24
Neutron so far has been relying on and evolving its own framework for	25
managing web service lifecycle and dispatch API operations to plugins.	26
27
Namely:	28
29
* API resource definition is performed using dictionaries, which contain	30
  information about object attribute types, default values and attribute	31
  validation. This has a number of limits, especially when it comes to	32
  performing validation and serialization of API resources, and it also	33
  encourages a behavior where everything is passed around as dictionaries.	34
35
* The current API extension management framework implies that extensions	36
  can pretty much do everything they want with the API - even redefining	37
  parts of it.	38
39
* There is home grown code based on fork() for managing multiple API workers.	40
  While this is generally not a problem, it still is a significant amount of	41
  code that needs to be maintained. Many REST frameworks like Pecan provide	42
  built-in support for spawning multiple API workers.	43
44
* The REST controllers have become heavyweight components since they also	45
  need to take care of tasks such as enforcing quotas and authorizing	46
  API requests.	47
48
* The actual response returned by the REST layer is currently built within	49
  the plugin, because there is no object-oriented interface with the plugin.	50
  Indeed, the REST controller passes the resource to the plugin as a dict,	51
  and expects a resource as a dict from the plugin. It assumes that the plugin	52
  builds a dictionary which respects the resource model (ensuring however	53
  that only valid attributes are returned to the API consumer).	54
55
* Most importantly, the current WSGI/REST framework is a relatively large	56
  size component in Neutron's codebase. Switching to a well-established	57
  framework will make the whole codebase a lot more maintainable.	58
59
Proposed Change	60
===============	61
62
In a nutshell: replace the existing framework with Pecan, remove the current	63
code, and ensure that operators are unaffected by the change.	64
65
This means that we expect the following for the Kilo release:	66
67
1) All API requests will be served by Pecan REST controllers	68
   This will have impact on in-tree attribute extensions. Such extensions	69
   indeed will be refactored as they currently define the resources they	70
   handle using the same dict-based style as attributes.py [3]_	71
   There will therefore be a new process for adding extensions to the Neutron	72
   API. This process will be documented as a part of this blueprint.	73
74
2) Service startup will not happen anymore through Python PasteDeploy, and	75
   will be managed by Pecan. Similarly multiple API workers will be handled	76
   through Pecan as well.	77
78
3) The Pecan REST controller will simply take care of serializing responses	79
   and deserializing requests into appropriate transfer objects describing	80
   API resources. However, the REST layer will no longer be responsible for	81
   authorization and quota enforcement. These operations will be handled by	82
   the new plugin layer discussed in the spec [2]_. For the sake of this	83
   document it is enough to say that these operations won't be performed by	84
   the plugin implementation.	85
   Request validation will occur in the REST API layer. The goal is to	86
   specify constraints using JSON schema. At the time of writing this spec it	87
   has not yet been analyzed whether it is possible to express all the	88
   validation constraints currently applied to the Neutron API using JSON	89
   schema. The final implementation, which is not necessarily the first	90
   iteration,  might either:	91
92
   * Use JSON schema only	93
94
   * Embed validation logic in API objects (see [2]_ more information)	95
96
   * Use a mix of JSON schema and custom validation logic, and possibly	97
     encapsulate everything within API objects.	98
99
4) On the other hand, authentication for a request must happen before the	100
   call is dispatched to the plugin layer. Pecan hooks [4]_ will be used to	101
   perform authentication at the appropriate time.	102
103
Assaf Muller	There's no explanation of this step anywhere in the spec. How will the …	Dec 16 11:24 PM
5) The Pecan framework however only takes care of managing the REST API	104
   server. For this reason as a part of this blueprint the REST and RPC over	105
   AMQP servers will be split. Potential impacts of this change on deployers	106
   are discussed in the relevant section. This split is not expected to have	107
   any other relevant impact on operators, developers, or users.	108
109
Data Model Impact	110
-----------------	111
112
No data model change expected.	113
114
REST API Impact	115
---------------	116
117
Even if this patch has a deep impact on the Neutron management layer, the REST	118
API itself will not change at all, and will preserve its capabilities in terms	119
of resources, available operations, filtering, pagination and sorting.	120
121
Security Impact	122
---------------	123
124
Radical changes in the framework handling REST API requests always have a	125
potential security impact.	126
127
In this case, since we are moving away from a home grown framework to one	128
which is already widely adopted across OpenStack projects, the overall	129
security level should increase.	130
131
Notifications Impact	132
--------------------	133
134
The REST API layer is currently responsible for sending notifications such as	135
those needed by the Telemetry service. With these change the notifications	136
will not be handled in the REST API layer anymore, but moved within the plugin	137
interface as specified in [2]_	138
139
Other End User Impact	140
---------------------	141
142
End users will not even notice the difference between a server running the home	143
grown framework and one which switched to Pecan.	144
145
Performance Impact	146
------------------	147
148
No significant impact expected.	149
We have however no measurement available to justify this claim.	150
151
For this reason performance measurements should be done as part of this	152
blueprint implementation to ensure that switching to Pecan does not	153
negatively impact application performance.	154
155
For the purpose of this work, Rally will be used to provide before/after	156
benchmarks.	157
If other tools such as OsProfiler are deemed useful, they will be used	158
as well in the evaluation.	159
160
IPv6 Impact	161
-----------	162
163
IPv6-related APIs and IPAM capabilities will be unchanged.	164
165
Other Deployer Impact	166
---------------------	167
168
We expect the deployer impact to be minimal.	169
The main difference introduced by this change, from a deployer perspective	170
is the fact that the HTTP server will be split from the AMQP server.	171
172
For green-field deployments this will not be a problem at all.	173
It will also provide deployers with the desirable option of deploying the	174
HTTP and AMQP servers on different nodes.	175
176
For existing deployments, updates should be smooth and transparent.	177
The only difference would be that after an upgrade there would not be	178
a single neutron server service, but two - one for the REST API, and one	179
for RPC over AMQP.	180
181
Developer Impact	182
----------------	183
184
New extensions will need to be developed in a different way.	185
This will be thoroughly documented in developer documentation.	186
187
Community Impact	188
----------------	189
190
Moving away from the home-grown framework will allow the community to focus	191
exclusively on Neutron's business logic. Moreover, members of the Neutron	192
community will also be encouraged to contribute back to Pecan.	193
194
Alternatives	195
------------	196
197
Other solutions such as Falcon [5]_ and WSME + Pecan [6]_ have been	198
considered. However the adoption of Pecan appears the one that better suits	199
Neutron.	200
201
A mailing list discussion [7]_ on REST API frameworks has been used to provide	202
some guidance. For WSME, even if it is an interesting solution to increase code	203
maintanability, and ease the development process, we struggled during some	204
early experiments to make it work with the current extension model. Even if	205
it might be argued that the problem in this case is the extension model, we are	206
unable to recommend it as a part of this blueprint.	207
208
209
Implementation	210
==============	211
212
Assignee(s)	213
-----------	214
215
Primary assignee:	216
  Mark McClain (markmcclain)	217
218
Other contributors:	219
  Sean Collins (sccal68) [developer docs]	220
  Salvatore Orlando (salv-orlando) [reserve dev]	221
222
Work Items	223
----------	224
225
1) Define framework for Pecan controllers for core and extended resources.	226
2) Re-implement controllers for base and extended resources, paying particular	227
   attention to dealing properly with 'attribute' extensions. The deliverable	228
   of this work item will be a new "base controller" which will leverage the	229
   v3 plugin interface proposed in [2]_.	230
3) Plug authorization and quota enforcement in the "plugin management"	231
   layer.	232
4) Split out RPC over AMQP server	233
5) Redefine unit tests to work with new framework	234
6) Validate new solution with integration testing, perform performance and	235
   scalability analysis.	236
237
Dependencies	238
============	239
240
* New plugin interface specification [2]_	241
242
Testing	243
=======	244
245
Once the changes are in place and integrated with the new plugin interface	246
discussed in [2]_, gate tests should run as usual. We do not expect this	247
change to have any impact that might trigger race conditions leading to	248
intermittent gate failures.	249
250
On the other hand, this change will have a significant impact on unit	251
testing. Most unit tests exercise the REST API server and with this change	252
these unit tests will be inevitably broken.	253
Under this proposal we therefore expect significant changes in the "base	254
classes" for unit test, such as [8]_.	255
256
257
Besides, new modules introduced as a part of this blueprint should be	258
thoroughly unit tested, with a target level of coverage between 90% and 100%.	259
Test coverage should be verified with tox -ecover.	260
261
Tempest Tests	262
-------------	263
264
No new tests are anticipated.	265
266
Functional Tests	267
----------------	268
269
Even if API functional testing will eventually be a relevant part of Neutron's	270
functional testing suite, this is outside the scope of this spec.	271
272
API Tests	273
---------	274
275
Please see previous section.	276
277
Documentation Impact	278
====================	279
280
As the specification discussed in this document changes the way in which the	281
Neutron server is deployed because of the split between the HTTP and RPC over	282
AMQP server, this will need to be appropriately documented in the admin guide.	283
284
User Documentation	285
------------------	286
287
No change.	288
289
Developer Documentation	290
-----------------------	291
292
The new process for developing Neutron extensions should be thoroughly	293
documented.	294
295
Also the developer documentation for the api layer [9]_ needs to be updated	296
according to the changes being made as part of this blueprint.	297
298
References	299
==========	300
			.. [1] Pecan documentation: http://pecan.readthedocs.org 	301
.. [2] v3 plugin interface: https://review.openstack.org/#/c/140527/ 	302
.. [3] https://github.com/openstack/neutron/blob/master/neutron/api/v2/attributes.py 	303
.. [4] Pecan hooks: http://pecan.readthedocs.org/en/latest/hooks.html 	304
.. [5] Falcon WSGI framework: http://falconframework.org/ 	305
.. [6] WSME: http://wsme.readthedocs.org/ 	306
.. [7] http://lists.openstack.org/pipermail/openstack-dev/2014-March/030385.html 	307
.. [8] http://git.openstack.org/cgit/openstack/neutron/tree/neutron/tests/unit/test_db_plugin.py 	308
.. [9] http://docs.openstack.org/developer/neutron/devref/api_layer.html 
