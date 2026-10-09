# SailPoint IdentityIQ Access Governance: RBAC, SoD, and Access Certification

**Week 11 of my 12 week IAM engineering curriculum. Project 3.**

I took an ERP application that nobody was governing, designed a role model for it, wrote segregation of duties rules, ran a manager access certification with three reviewers, removed the access they rejected, and pulled the audit evidence to prove it. Everything here was built by hand on a local SailPoint IdentityIQ 8.3 instance, on top of the HR source from Week 5 and the Joiner and Leaver workflows from Week 6.

Related repos: [Week 6 JML workflows](https://github.com/GokuCooper/sailpoint_iiq_jml_workflows)

## The short version

| Measure | Before | After |
|---|---|---|
| Roles defined | 0 | 10 (5 business, 5 IT) |
| SoD policies | 0 | 2 policies, 3 rules |
| Policy violations | Unknown, no rules existed | 3 found, 3 fixed, 0 remaining |
| ERP accounts | 13 | 11 |
| Leaver with live ERP access | 1 | 0 |
| Orphaned accounts | 1 | 0 |
| Access reviews completed | none | 3 of 3 |
| Certification items decided | none | 24 of 24 (17 approved, 7 revoked) |

The whole campaign, from launch to the last manager signing, took 29 minutes. Access was removed and verified about 6 minutes after that. These are lab numbers on test data with one person playing every role, so they show the process works. They are not a production benchmark.

## What these words mean

I wrote these the way I would explain them to a 5th grader, because if I cannot explain it simply I do not understand it yet.

| Term | Plain English |
|---|---|
| RBAC (role based access control) | The school does not hand every kid a different pile of hall passes. It makes a few badges, like Library Helper. You get the badge that fits your job, and the badge comes with the passes. |
| Business role | The badge. It says who you are, like AP Clerk. |
| IT role | The key ring. It bundles the real keys from the system. |
| Entitlement | One single key. |
| Birthright role | The library card every student gets on day one. |
| SoD (segregation of duties) | The kid who collects the bake sale money should not be the same kid who writes down how much was collected. |
| Access certification | Roster check day. Each manager reads the list of what their people can do and says keep or take away for every line. |
| Remediation | Taking the key back after the manager says take away. |
| Audit evidence | Showing your work so someone else can check it. |

## Environment

| Component | Detail |
|---|---|
| Platform | SailPoint IdentityIQ 8.3 on Apache Tomcat 9 and MySQL 8, local install |
| Authoritative source | HR_Employee_Feed, a delimited file application from Week 5 |
| Governed application | DKN_ERP, a delimited file application built this week |
| Existing automation | DKN Joiner and DKN Leaver lifecycle events from Week 6, provisioning to DKN_Portal |
| Population | 12 HR identities, 3 of them managers |
| Access data | 13 ERP accounts, 7 entitlements, 26 entitlement assignments |

## Situation

After Weeks 5 and 6, IdentityIQ knew who my people were and could create and disable a portal account for them. It knew nothing about what they could do inside a finance system. There were no roles, no rules, and no review. Access in the ERP file was a pile of loose keys.

To make the project a real test, I seeded five problems into the ERP data on purpose, the way a control tester plants known defects to see if a control catches them. The goal was to find all five through the tools, without fixing anything by hand first.

| Seeded finding | Who | Why it is bad |
|---|---|---|
| Can create vendors and approve payments | Nia Carter | One person can pay a fake vendor |
| User administration plus payroll | Caleb Okafor | One person can create and pay a ghost employee |
| User administration plus audit log review | Marcus Reed | Grading his own homework |
| Leaver still has an active account | Jordan Hayes | Inactive in HR, still holding keys |
| Account with no matching employee | kevin.marsh | Orphaned account nobody owns |

## Build

### Phase 1: Capture the before state

Six identities, empty role columns, zero roles, zero policies.

![Identity Warehouse baseline](screenshots/01_identity_warehouse_baseline.png)

![Policies page before any SoD rules](screenshots/03_policies_before_sod.png)

### Phase 2: Grow the HR population from 5 to 12

Five people report to one manager, which does not make much of a certification. I added 7 new hires to the HR feed so there would be three managers and three separate reviews. The HR aggregation scanned 12 accounts, created 7 identities, and updated 5.

This phase produced two of the snags below, because my own Week 6 guardrail stopped the batch.

![Refresh blocked by the Joiner threshold](screenshots/05_refresh_failed_joiner_threshold_exceeded.png)

![Managers linked after the refresh](screenshots/10_identity_warehouse_12_identities_managers_linked.png)

### Phase 3: Onboard the ERP application

DKN_ERP is a delimited file with one row per access group, so a person with four keys has four rows. Three settings made it work:

- **Merging** on `accountId` with `accessGroup` as the merged column, so each person's rows stack into one account.
- **Schema** with `accessGroup` flagged as Managed, Entitlement, and Multi Valued.
- **Correlation** from the account `email` to the identity `email`, because two people can share a name and nobody shares a work email.

![Schema with the entitlement flags](screenshots/18_erp_application_schema_entitlement_flag.png)

The aggregation result confirmed every number in the file: 13 accounts scanned, 12 matched to existing people, 1 new placeholder identity for the orphan, 7 entitlements promoted to the catalog, and 26 identity entitlements created. That 26 is the proof that merging worked. Without it the count would have been 13.

![ERP aggregation result](screenshots/22_erp_aggregation_13_scanned_26_entitlements.png)

This is the before picture for the whole project. Nia Carter holds four keys, including the toxic pair, and IdentityIQ has no opinion about it.

![Nia Carter before governance](screenshots/25_nia_carter_entitlements_before_governance.png)

### Phase 4: Design and build the RBAC model

Ten roles. Five IT roles bundle the entitlements, and five business roles assign them based on HR attributes.

| Business role | Who gets it | Required IT role | Entitlements granted |
|---|---|---|---|
| All Employees (birthright) | Employment Status equals Active | ERP Base Access | ERP_Base_User |
| AP Clerk | Job Title equals Accounts Payable Specialist | AP Invoice Entry | AP_Create_Vendor and AP_Enter_Invoice |
| AP Manager | Job Title equals Accounts Payable Manager | AP Payment Approval | AP_Approve_Payment |
| Payroll Specialist | Job Title equals Payroll Specialist | Payroll Processing | PAY_Edit_Payroll |
| IT Security Admin | Job Title equals Systems Administrator | User Administration | ERP_User_Admin |

ERP_Audit_Log_Review is left out of every role on purpose. It stays a loose key so the certification has something outside the role model to judge.

AP Invoice Entry needs two entitlements. The simple view only shows a list, so I opened the advanced view to confirm the logic was "contains all" and not "contains any."

![Advanced view showing containsAll](screenshots/27_it_role_ap_invoice_entry_advanced_view_and_logic.png)

![Role viewer with all 10 roles](screenshots/31_role_viewer_10_roles_ap_clerk_rule_and_required_role.png)

I built a separate DKN Governance Refresh task with one option checked, so role detection would not trigger my Week 6 lifecycle events. After it ran, the two role columns that were empty in the morning were filled in.

![Assigned and detected roles](screenshots/34_identity_warehouse_roles_assigned_and_detected.png)

Assigned is the badge IdentityIQ says you should have. Detected is the key ring it found on you. On Nia's record, the Allowed By column explains two of her three key rings. The third, AP Payment Approval, is allowed by nothing.

![Nia Carter with an unexplained role](screenshots/35_nia_carter_roles_after_rbac.png)

### Phase 5: Write the SoD policies

| Rule | Policy type | Left side | Right side |
|---|---|---|---|
| AP Entry vs AP Approval | Role SoD | AP Clerk, AP Invoice Entry | AP Manager, AP Payment Approval |
| Payroll vs User Administration | Role SoD | Payroll Specialist, Payroll Processing | IT Security Admin, User Administration |
| User Admin vs Audit Log Review | Entitlement SoD | ERP_User_Admin | ERP_Audit_Log_Review |

I used both policy types because they see different things. Role SoD watches badges and key rings. Entitlement SoD watches single keys, and the audit log key belongs to no role, so only an entitlement rule can see it.

Each rule carries correction advice and a compensating control, and violations are owned by the person's manager.

![Both role rules simulated](screenshots/40_role_sod_simulation_both_rules_2_identities.png)

![Two active SoD policies](screenshots/44_policies_list_two_active_sod_policies.png)

After the fix described in Snag 6, the policy check recorded three violations on three people, and a clean control identity stayed clean.

![Three violations after the fix](screenshots/56_governance_refresh_3_violations_after_fix.png)

![Nia Carter's violation with advice attached](screenshots/47_nia_carter_policy_violation.png)

### Phase 6: Run the access certification

I gave the three managers passwords and ran a Manager certification for all managers, scoped to DKN_ERP, including roles, additional entitlements, and policy violations.

Settings I changed from the defaults, and why:

| Setting | Changed to | Reason |
|---|---|---|
| Require Comments For Revocation | On | Every take away decision carries a written reason |
| Enable Account Revocation | On | A leaver should lose the whole account, not one key |
| Enable Bulk Approval | Off | Stops a reviewer from approving everything in one click |
| Suppress Initial Notifications | On | The lab has no mail server |

![Behavior settings](screenshots/62_manager_certification_behavior.png)

The campaign launched with 3 access reviews, 11 identities, and 24 items, all at 0 percent.

![Campaign at launch](screenshots/64_certification_launched_3_reviews_24_items_baseline.png)

A manager who signs in sees only Home and My Work. No Setup, no Applications. She can act on her people and cannot change how the system is configured.

![Manager home page with 3 violations waiting](screenshots/58_priya_shah_manager_login_home.png)

Derrick Coleman and Simone Whitfield had clean reviews and approved 5 items between them. Priya Shah's review held all the findings. IdentityIQ put her three violations on an Important tab ahead of everything else.

![Violations surfaced first](screenshots/74_priya_access_review_important_tab_3_violations.png)

Correcting a violation asks which side of the conflict to remove. Picking the wrong side would take away the access the person needs for the job and leave the risky one in place.

![Choosing what to remove for Nia](screenshots/77_priya_correct_violation_nia_remove_ap_payment_approval.png)

The comment is required, and the recipient shows who has to do the work. Because DKN_ERP is a file, IdentityIQ cannot remove access by itself, so it routes a task to the application owner.

![Required comment and recipient](screenshots/79_priya_revocation_comment_and_recipient.png)

All 19 of Priya's decisions before sign off:

![Review tab part 1](screenshots/81_priya_review_tab_decisions_part_1.png)

![Review tab part 2](screenshots/82_priya_review_tab_decisions_part_2.png)

The same entitlement got two different answers. ERP_Audit_Log_Review was revoked for Marcus Reed, a Systems Administrator, and approved for Renee Dawson, the Internal Auditor. That is the reason a human reviews access and a script does not.

| Reviewer | Signed | Items |
|---|---|---|
| Derrick Coleman | 4:59:58 PM | 2 approved |
| Simone Whitfield | 5:00:55 PM | 3 approved |
| Priya Shah | 5:24:21 PM | 12 approved, 4 revoked, 3 violations corrected |

![Campaign complete](screenshots/85_certification_complete_3_of_3_reviews_24_of_24_items.png)

### Phase 7: Remediate and prove it

Signing produced one remediation work item for the application owner, with four removals named exactly.

![Remediation work item](screenshots/87_remediation_work_item_4_removals.png)

I removed those four rows from the ERP file and nothing else. The before and after files are both in the `evidence` folder. Then I turned on Detect deleted accounts, so IdentityIQ would notice that Jordan's account no longer existed, and aggregated.

![Aggregation after remediation](screenshots/89_erp_aggregation_after_remediation_1_link_deleted.png)

The next governance refresh reported no violations. Nia's record now shows four roles, every one explained in the Allowed By column, three entitlements, and no policy violations.

![Nia Carter after remediation](screenshots/91_nia_carter_entitlements_after_remediation.png)

![No policy violations](screenshots/92_nia_carter_no_policy_violations.png)

Nia's record at three points in the day:

| Screenshot | Roles | Entitlements | Violations |
|---|---|---|---|
| 25, before | None | 4, including the toxic pair | None, because no rule existed |
| 35, middle | 5, one with nothing allowing it | 4 | 1 once policies ran |
| 91, after | 4, all explained | 3 | None, because the conflict is gone |

The manager's violation count went from 3 to 0.

![Manager home page with zero violations](screenshots/95_priya_home_zero_policy_violations.png)

The orphan account sat outside the campaign, since kevin.marsh has no manager to review him. I removed it as a separate application owner decision and aggregated again: 11 accounts scanned, 1 link deleted.

### Phase 8: Pull the audit evidence

| Report | What it proves | Screenshot | Exported file in `evidence` |
|---|---|---|---|
| Access Review Signoff Live Report | Who signed each review and when | 97 | `dknQ42026AccessReviewSignoff.pdf` and `.csv` |
| Access Review Decision Report | Every decision: 17 approved, 7 revoked, 0 exceptions, 0 undecided | 98 | `dknQ42026AccessReviewDecisions.pdf` |
| Revocation Live Report | Each revocation and its status, ending at 7 of 7 Finished | 99, 102, and 104 to 106 | `dknQ42026Revocations.pdf` and `.csv`, plus the October 9 rerun |
| Policy Violation Report | Zero current violations | 100 | `dknPolicyViolationsCurrent.pdf` and `.csv` |

![Decision report](screenshots/98_report_access_review_decisions.png)

![Policy violation report with no results](screenshots/100_report_policy_violations_none.png)

## Snags

This is the part I would talk about in an interview. For each one I name the layer the problem lived in before describing the fix.

### 1. My own guardrail stopped the batch

**What happened:** The identity refresh failed with "7 of the 13 identities processed by this task has resulted in Lifecycle Event DKN Joiner. A fixed threshold value of 5 has been met."

**Root cause layer:** Configuration, working as designed. In Week 6 I set an identity processing threshold of 5 on the Joiner so a corrupted HR file could not mass provision accounts.

**Fix:** I validated the batch against the aggregation result, which showed exactly 7 created. Then I raised the threshold to 10, ran the refresh, and set it back to 5. All three states are captured.

![Threshold restored](screenshots/09_joiner_threshold_restored_to_5.png)

**Lesson:** The refresh task has a "Disable Identity Processing Threshold" switch that would have turned the safety off for every event at once. A scoped, documented, reversed override on one event is the better change.

### 2. Events processed, but no account

**What happened:** The refresh said 7 events were processed, yet Elena Vasquez had no portal account.

**Root cause layer:** Process timing and visibility, not a failure.

**How I worked it:** I checked three things in order: task results for the workflow, work items waiting on a human, and a known good identity from Week 6. My own task history showed the pattern. On September 23 the Joiner ran at 8:10 PM and the portal aggregation ran at 8:15 PM. The Joiner writes the account into the portal. IdentityIQ cannot see it until the next aggregation reads it back. The Joiner also runs in the background about five minutes after the refresh.

**Proof:** Elena's portal account shows `created_at` of 3:25 PM, the same minute as her Joiner task result.

![Elena's portal account](screenshots/16_elena_vasquez_portal_account_confirmed.png)

**Lesson:** Provisioning and visibility are two separate steps. "Events processed" means the work was handed off, not that it finished.

### 3. Underscores are not allowed in task names

**What happened:** Saving `DKN_ERP Account Aggregation` failed with "Character '_' is not allowed in object names," even though application names accept underscores.

**Root cause layer:** Naming rules that differ by object type.

**Fix:** Renamed with spaces, and used spaces for every role, policy, task, and report name after that.

### 4. Two different things called "rule"

**What happened:** While building the first SoD policy I opened a Rule Editor and started typing the conflict pair into it. That editor is for a script that decides who owns a violation. The conflict pairs live further down the page under SOD Policy Rules.

**Root cause layer:** User interface. The same word means two different objects on one screen.

**Fix:** I caught it before saving, cancelled, set the violation owner to the manager, and used Create New Rule.

![The wrong editor](screenshots/36_wrong_rule_editor_opened_policy_owner_script.png)

### 5. A badge only rule would have missed the real finding

**What happened:** My first design for the AP rule was AP Clerk versus AP Manager. After the roles were detected, the data showed why that would fail. Nia does not have the AP Manager badge. She has the AP Payment Approval key ring.

**Root cause layer:** Control design.

**Fix:** Each side of each rule lists both the business role and the IT role. The rule now blocks someone being assigned two conflicting jobs and also catches someone quietly holding the keys.

**Lesson:** Design the control against how access is really held, not how the org chart says it should be.

### 6. The simulation passed and the control was off

This is the one I learned the most from.

**What happened:** All three rule simulations returned 1 violation each. I ran the real policy check expecting 3. It reported 2. Marcus Reed showed no violations, two minutes after a simulation said he had one.

![Two violations where three were expected](screenshots/46_governance_refresh_2_violations_expected_3.png)

**How I worked it:** I did not rerun or rebuild anything. I checked three layers in order:

| Check | Result | What it ruled out |
|---|---|---|
| Does Marcus hold both entitlements? | Yes | The data |
| Is the saved policy active with the rule listed? | Yes | The policy |
| Is the saved rule intact? | Both sides intact, but Disabled was checked | Found it |

![Root cause](screenshots/53_root_cause_rule_disabled_checkbox_checked.png)

**Root cause layer:** Policy rule configuration. Disabled was unchecked in my screenshot from before the simulation and checked afterward, and the only action in between was Run Simulation on that rule. I saw the same behavior at the policy level, where a simulation flipped State to Inactive. The simulation still reported 1 violation while the rule was disabled, so it tests a rule whether or not it is switched on. The Policies list said Active the whole time, which was true for the policy and hid the state of the rule inside it.

**Fix:** Unchecked Disabled, confirmed State was Active, saved, and reran. Three violations.

**Lesson:** A passing test does not prove a control is enabled. Here the test tool changed the state of the thing it tested. I only caught it because I had an expected count of 3 and did not accept 2. After any simulation I now check State on the policy and Disabled on every rule before saving.

### 7. Access was gone, but the revocation report took two days to agree

**What happened:** The Revocation Live Report lists 7 lines. Twenty minutes after I completed the work item, 3 read Finished and 4 read Open. I ran it twice that day with the same result.

| Lines | Status on October 7 |
|---|---|
| 3 role and entitlement lines IdentityIQ linked to the violations | Finished |
| 3 violation lines and Jordan's ERP Base Access, the same 4 that went into the work item | Open |

![Revocation report, second run](screenshots/102_report_revocations_3_finished_4_open_second_run.png)

**Root cause layer:** Verification bookkeeping, not access.

**How I worked it:** I did not reopen the work item or touch the certification. I checked the source of truth first. The aggregation had deleted the rows, the refresh found zero violations, and the Policy Violation report returned nothing, so the access was gone. Then I looked for the mechanism. The Perform Maintenance task has "Scan for completed revocations" enabled, which told me IdentityIQ had its own background check for this and that the right move was to wait and rerun.

![Background scan enabled](screenshots/103_perform_maintenance_scan_for_completed_revocations.png)

**Result:** I reran the report two days later, on October 9. All 7 lines read Finished, with no further action from me.

![Revocation report two days later](screenshots/104_report_revocations_rerun_two_days_later.png)

The report does not fit on one screen, so the remaining lines are in screenshots 105 and 106.

**What I still do not know:** Exactly when the status flipped. It happened somewhere between 20 minutes and 2 days after the work item was completed, so the scan runs on a slower cycle than the five minute maintenance loop I saw with the Joiner.

**Lesson:** A revocation has three stages. The manager decides, someone executes, and the system verifies. Completing a work item by hand closes the second stage, and IdentityIQ closes the third on its own schedule. When a status column and the source system disagree, the source system is the evidence, and the status column deserves a second look later before anyone forces it.

## Result

| Seeded finding | Caught by | Outcome |
|---|---|---|
| Nia Carter: vendor entry plus payment approval | Role SoD | Revoked in certification, verified removed |
| Caleb Okafor: user administration plus payroll | Role SoD | Revoked in certification, verified removed |
| Marcus Reed: user administration plus audit log review | Entitlement SoD | Revoked in certification, verified removed |
| Jordan Hayes: leaver with an active ERP account | Manager certification | Account removed |
| kevin.marsh: orphaned account | Correlation during aggregation | Removed by the application owner |

Five seeded, five found, five closed. No clean identity was flagged by any rule.

Jordan's record shows two controls from two different weeks landing on the same person. His portal account was disabled by the Week 6 Leaver on September 23, and his ERP account was removed by this certification.

![Jordan Hayes with no ERP account](screenshots/94_jordan_hayes_erp_account_removed.png)

## Real World Mapping

| What I did in the lab | What it is at a company |
|---|---|
| Seeded five known defects before building controls | Control testing with planted exceptions |
| Raised a threshold, ran the batch, restored it, with screenshots | Change management for a temporary control override |
| Business roles assigned from HR job title | Birthright and job based provisioning |
| Role SoD and entitlement SoD rules with correction advice | Preventive and detective SoD controls for SOX |
| Manager certification with required revocation comments | Quarterly user access review |
| Bulk approval disabled | A guard against rubber stamp reviews |
| Remediation work item routed to the application owner | Manual fulfillment for a disconnected application |
| Removing rows, aggregating, and checking violations | Closed loop remediation and verification |
| Signoff, decision, revocation, and violation reports | The evidence package an auditor asks for |

## Known gaps and next steps

- **Nobody reviewed the top manager or the orphan.** A manager certification only covers people who have a manager. Priya Shah and kevin.marsh were both outside it. In production I would add an application owner certification and a scheduled uncorrelated account report.
- **Verification lag on manual remediation.** The revocation report took up to two days to show all 7 lines Finished. See Snag 7. In production I would agree a verification window with audit up front.
- **Remediation was manual.** DKN_ERP is a read only file. A real connector with provisioning would let IdentityIQ remove access itself.
- **One person played every part.** I was the administrator, all three managers, and the application owner. The separation between those jobs is real in the tool and simulated in the lab.
- **Role mining is not covered.** I designed the roles top down from job titles. The Role Mining tab is the natural next exercise.

## Resume bullets

- Designed and built a 10 role RBAC model (5 business, 5 IT) in SailPoint IdentityIQ 8.3, assigning access from HR job title and employment status across 12 identities and 26 entitlement assignments.
- Authored 3 segregation of duties rules across role and entitlement policy types, detecting 3 toxic access combinations with zero false positives against seeded test data.
- Ran a manager access certification across 3 reviewers, 11 identities, and 24 items to 100 percent completion, then remediated and verified 7 revocations, reducing open SoD violations from 3 to 0 and removing 1 leaver account and 1 orphaned account.
- Diagnosed a silently disabled SoD rule by comparing expected and actual violation counts, isolating the cause across data, policy, and rule layers before making any change.

## The interview story

**Problem.** An ERP application had 13 accounts and no governance. Nobody could say who should have what, and nothing would notice if one person could create a vendor and also pay it.

**Solution.** I onboarded the application, built 10 roles from HR attributes, wrote 3 SoD rules, and ran a manager certification that put the violations in front of the right manager with written advice on how to fix each one.

**Impact.** All five seeded problems were found and closed. Three violations went to zero, a leaver lost access, and an orphan account was removed, with four reports to prove it.

**Learning.** A simulation told me a control worked while the control was switched off. I caught it because I knew the number I expected. I now treat a passing test and an enabled control as two separate facts and check both.

## Repository layout

```
sailpoint_iiq_access_governance/
├── README.md
├── screenshots/        107 images, numbered 01 to 106 plus 96b
└── evidence/
    ├── hr_feed.csv
    ├── erp_accounts_before_remediation.csv
    ├── erp_accounts.csv
    ├── dknQ42026AccessReviewSignoff.pdf and .csv
    ├── dknQ42026AccessReviewDecisions.pdf
    ├── dknQ42026Revocations.pdf and .csv
    └── dknPolicyViolationsCurrent.pdf and .csv
```

All names, accounts, and email addresses are fictional test data on the `iamlab.local` lab domain.

## Full screenshot index

| # | File | Shows |
|---|---|---|
| 01 | `01_identity_warehouse_baseline.png` | Starting identities, empty role columns |
| 02 | `02_role_viewer_before_rbac.png` | Zero roles |
| 03 | `03_policies_before_sod.png` | Zero policies |
| 04 | `04_hr_aggregation_12_scanned_7_created.png` | HR feed grown to 12 |
| 05 | `05_refresh_failed_joiner_threshold_exceeded.png` | Guardrail blocks the batch |
| 06 | `06_joiner_threshold_original_value_5.png` | Threshold before |
| 07 | `07_joiner_threshold_raised_to_10.png` | Threshold during |
| 08 | `08_refresh_success_7_events_processed.png` | Refresh succeeds |
| 09 | `09_joiner_threshold_restored_to_5.png` | Threshold restored |
| 10 | `10_identity_warehouse_12_identities_managers_linked.png` | Three managers linked |
| 11 | `11_elena_vasquez_no_portal_account_after_joiner.png` | Unexpected missing account |
| 12 | `12_task_results_week6_joiner_then_aggregate_pattern.png` | Week 6 pattern in task history |
| 13 | `13_nia_carter_known_good_portal_account.png` | Known good comparison |
| 14 | `14_task_results_joiner_ran_for_7_new_hires.png` | Seven Joiner runs |
| 15 | `15_portal_aggregation_9_scanned_7_updated.png` | Portal accounts become visible |
| 16 | `16_elena_vasquez_portal_account_confirmed.png` | Account confirmed, timestamps match |
| 17 | `17_erp_merging_configuration.png` | Merging settings |
| 18 | `18_erp_application_schema_entitlement_flag.png` | Schema and entitlement flags |
| 19 | `19_erp_correlation_email_to_identity.png` | Correlation |
| 20 | `20_task_name_underscore_not_allowed_error.png` | Naming error |
| 21 | `21_erp_aggregation_task_promote_managed_attributes.png` | Aggregation task options |
| 22 | `22_erp_aggregation_13_scanned_26_entitlements.png` | ERP aggregation result |
| 23 | `23_entitlement_catalog_erp.png` | Seven entitlements |
| 24 | `24_kevin_marsh_uncorrelated_orphan_account.png` | Orphan identified |
| 25 | `25_nia_carter_entitlements_before_governance.png` | Before picture |
| 26 | `26_it_role_ap_invoice_entry_entitlement_profile.png` | IT role profile |
| 27 | `27_it_role_ap_invoice_entry_advanced_view_and_logic.png` | containsAll logic |
| 28 | `28_role_viewer_5_it_roles.png` | Five IT roles |
| 29 | `29_business_role_ap_clerk_assignment_rule.png` | Assignment rule |
| 30 | `30_business_role_all_employees_birthright.png` | Birthright role |
| 31 | `31_role_viewer_10_roles_ap_clerk_rule_and_required_role.png` | All 10 roles |
| 32 | `32_governance_refresh_task_role_option.png` | Governance refresh options |
| 33 | `33_governance_refresh_result.png` | Roles applied |
| 34 | `34_identity_warehouse_roles_assigned_and_detected.png` | Role columns filled |
| 35 | `35_nia_carter_roles_after_rbac.png` | Middle picture |
| 36 | `36_wrong_rule_editor_opened_policy_owner_script.png` | Wrong editor |
| 37 | `37_role_sod_rule_ap_entry_vs_ap_approval.png` | First SoD rule |
| 38 | `38_role_sod_simulation_ap_rule_result.png` | First simulation |
| 39 | `39_role_sod_rule_payroll_vs_user_administration.png` | Second SoD rule |
| 40 | `40_role_sod_simulation_both_rules_2_identities.png` | Both rules simulated |
| 41 | `41_role_sod_policy_two_rules_active.png` | Role policy active |
| 42 | `42_entitlement_sod_rule_user_admin_vs_audit_log.png` | Entitlement rule |
| 43 | `43_entitlement_sod_simulation_1_violation.png` | Entitlement simulation |
| 44 | `44_policies_list_two_active_sod_policies.png` | Two active policies |
| 45 | `45_governance_refresh_check_active_policies.png` | Policy check enabled |
| 46 | `46_governance_refresh_2_violations_expected_3.png` | Unexpected count |
| 47 | `47_nia_carter_policy_violation.png` | Nia's violation |
| 48 | `48_caleb_okafor_policy_violation.png` | Caleb's violation |
| 49 | `49_marcus_reed_no_violation_unexpected.png` | Missing violation |
| 50 | `50_elena_vasquez_no_violations_control.png` | Clean control |
| 51 | `51_marcus_reed_entitlements_both_keys_confirmed.png` | Data ruled out |
| 52 | `52_entitlement_sod_policy_saved_looks_healthy.png` | Policy ruled out |
| 53 | `53_root_cause_rule_disabled_checkbox_checked.png` | Root cause |
| 54 | `54_simulation_passes_while_rule_disabled.png` | Simulation ignores Disabled |
| 55 | `55_rule_disabled_unchecked_fix.png` | Fix |
| 56 | `56_governance_refresh_3_violations_after_fix.png` | Three violations |
| 57 | `57_marcus_reed_policy_violation_after_fix.png` | Marcus's violation |
| 58 | `58_priya_shah_manager_login_home.png` | Manager view, 3 violations |
| 59 | `59_manager_certification_basic_settings.png` | Campaign basics |
| 60 | `60_manager_certification_lifecycle.png` | Lifecycle |
| 61 | `61_manager_certification_notifications.png` | Notifications |
| 62 | `62_manager_certification_behavior.png` | Behavior |
| 63 | `63_manager_certification_advanced.png` | Advanced |
| 64 | `64_certification_launched_3_reviews_24_items_baseline.png` | Campaign at 0 percent |
| 65 | `65_derrick_coleman_home_one_access_review.png` | Reviewer home |
| 66 | `66_derrick_my_access_reviews_list.png` | Review list |
| 67 | `67_derrick_access_review_before_decisions.png` | Clean review |
| 68 | `68_derrick_access_review_decisions_selected.png` | Decisions marked |
| 69 | `69_derrick_sign_off_attestation_prompt.png` | Attestation prompt |
| 70 | `70_derrick_access_review_signed_off_empty_queue.png` | Signed |
| 71 | `71_simone_home_one_access_review.png` | Second reviewer |
| 72 | `72_simone_access_review_3_decisions_selected.png` | Decisions marked |
| 73 | `73_simone_access_review_signed_off_empty_queue.png` | Signed |
| 74 | `74_priya_access_review_important_tab_3_violations.png` | Violations first |
| 75 | `75_priya_correct_violation_caleb_remove_payroll_processing.png` | Correction for Caleb |
| 76 | `76_priya_correct_violation_marcus_remove_audit_log_review.png` | Correction for Marcus |
| 77 | `77_priya_correct_violation_nia_remove_ap_payment_approval.png` | Correction for Nia |
| 78 | `78_priya_three_violations_marked_revoke.png` | Three marked |
| 79 | `79_priya_revocation_comment_and_recipient.png` | Required comment |
| 80 | `80_priya_open_tab_13_items_jordan_revoke_marked.png` | Remaining 13 |
| 81 | `81_priya_review_tab_decisions_part_1.png` | Decisions, part 1 |
| 82 | `82_priya_review_tab_decisions_part_2.png` | Decisions, part 2 |
| 83 | `83_priya_access_review_signed_off_empty_queue.png` | Signed |
| 84 | `84_priya_home_violations_still_3_until_remediated.png` | Decided, not yet fixed |
| 85 | `85_certification_complete_3_of_3_reviews_24_of_24_items.png` | Campaign at 100 percent |
| 86 | `86_spadmin_remediation_work_item_from_priya.png` | Work item arrives |
| 87 | `87_remediation_work_item_4_removals.png` | Four removals |
| 88 | `88_erp_aggregation_detect_deleted_accounts.png` | Detect deleted accounts |
| 89 | `89_erp_aggregation_after_remediation_1_link_deleted.png` | Aggregation after fix |
| 90 | `90_governance_refresh_zero_violations.png` | Zero violations |
| 91 | `91_nia_carter_entitlements_after_remediation.png` | After picture |
| 92 | `92_nia_carter_no_policy_violations.png` | No violations |
| 93 | `93_remediation_work_item_completed.png` | Work item closed |
| 94 | `94_jordan_hayes_erp_account_removed.png` | Leaver account gone |
| 95 | `95_priya_home_zero_policy_violations.png` | Manager view, 0 violations |
| 96 | `96_erp_aggregation_orphan_account_removed.png` | Orphan removed |
| 96b | `96b_reports_list_access_review_category.png` | Report catalog |
| 97 | `97_report_access_review_signoff.png` | Signoff report |
| 98 | `98_report_access_review_decisions.png` | Decision report |
| 99 | `99_report_revocations_status_in_progress.png` | Revocation report, first run |
| 100 | `100_report_policy_violations_none.png` | Violation report, empty |
| 101 | `101_certification_final_state.png` | Campaign final state |
| 102 | `102_report_revocations_3_finished_4_open_second_run.png` | Revocation report, second run |
| 103 | `103_perform_maintenance_scan_for_completed_revocations.png` | Background scan enabled |
| 104 | `104_report_revocations_rerun_two_days_later.png` | Rerun, Caleb Okafor and Jordan Hayes lines |
| 105 | `105_report_revocations_rerun_marcus_lines_finished.png` | Rerun, Marcus Reed lines |
| 106 | `106_report_revocations_rerun_nia_lines_finished.png` | Rerun, Nia Carter lines |
