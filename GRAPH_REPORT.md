# Graph Report — Minilab Codebase
<!-- Use: awk '/^## §God-Nodes/,/^## §/' docs/startup/GRAPH_REPORT.md | head -n -1 -->

## §Summary
- 1902 nodes · 2848 edges · 135 communities detected
- Extraction: 100% EXTRACTED · 0% INFERRED · 0% AMBIGUOUS · INFERRED: 6 edges (avg confidence: 0.8)
- Token cost: 0 input · 0 output

## §God-Nodes
1. `POST()` - 46 edges
2. `GET()` - 38 edges
3. `tgSend()` - 14 edges
4. `executeFormSubmission()` - 14 edges
5. `processClawBotMessage()` - 14 edges
6. `advanceStep()` - 12 edges
7. `getConnection()` - 12 edges
8. `isConnected()` - 12 edges
9. `callMCP()` - 12 edges
10. `tgSend()` - 11 edges

## §Connections
- `GET()` --calls--> `ensureBalances()`  [EXTRACTED]
  app\api\webhooks\whatsapp\route.ts → app\api\app\leave\route.ts
- `GET()` --calls--> `detectStaffRole()`  [EXTRACTED]
  app\api\webhooks\whatsapp\route.ts → app\api\app\session\route.ts
- `GET()` --calls--> `toCsv()`  [EXTRACTED]
  app\api\webhooks\whatsapp\route.ts → app\api\bm\export\route.ts
- `GET()` --calls--> `nextDay()`  [EXTRACTED]
  app\api\webhooks\whatsapp\route.ts → app\api\bm\reports\daily\route.ts
- `GET()` --calls--> `addDays()`  [EXTRACTED]
  app\api\webhooks\whatsapp\route.ts → app\api\bm\reports\daily\route.ts

## §Hyperedges
- **AI Pipeline Model Routing** — minilab_process_message, deepseek_routing, s40_threelayerarch, concept_two_stage_ai_routing [EXTRACTED 0.95]
- **Session Startup Protocol** — CLAUDE_minilab_platform, startup_quick_context, startup_recent_decisions, startup_v20_handover [EXTRACTED 1.00]
- **Auth Channel Matrix** — s6_whatsapp_otp, s6_telegram_botlogin, portal_routing [EXTRACTED 0.92]
- **AI Safety Stack** — s28_6hardrules, s40_hallucinationguard, concept_require_permission [INFERRED 0.85]
- **Portal Completeness Audits** — audit_resident_domain, audit_superadmin_bm, audit_permissions_rls, capabilities_map [INFERRED 0.85]

## §Community-Index

| Community # | Name | Key Functions | Cohesion |
|-------------|------|---------------|----------|
| 1 | Shared UI Components | ctxRoleLabel(), ctxSubtitle(), handleOpen(), loadNotifications(), handleClose(), handleDone(), handleSubmit(), reset() | 0.01 |
| 2 | AI Pipeline & Sync | addDays(), notifyContractorInvoice(), run(), getBrandingForBuilding(), getPmcBranding(), savePmcBranding(), getBmTelegramIds(), logPdpaAction() (+55 more) | 0.02 |
| 3 | AI Providers & Tool Registry | AnthropicProvider, generateEmbedding(), searchDocuments(), buildSchemaPrompt(), convertQuestionToQuery(), advanceConversationFlow(), clearConversationState(), getConversationState() (+26 more) | 0.02 |
| 4 | Attendance & Data Import | addDays(), areasKey(), avg(), broadcast(), buildingHasPmc(), checkHealth(), cleanName(), computeStatus() (+78 more) | 0.03 |
| 5 | AI Auto-Reply & Citations | buildCitationContext(), buildCitationInstruction(), captionImage(), processImage(), compileMemory(), validateOutput(), detectLanguageFallback(), detectUrgencyFallback() (+34 more) | 0.04 |
| 6 | Admin Tool Execution | cleanTimeInput(), expandDayRange(), parseDayName(), splitTimeRange(), adaptResponse(), stripMarkdown(), executeAddAgmMotion(), executeApproveRenovation() (+12 more) | 0.04 |
| 7 | Auto-Registration & Cache | continueRegistration(), handleAutoRegistration(), REGISTRATION_KEY(), handleStageAction(), runEngine(), handleMaterialReply(), sendReply(), fallbackParse() (+23 more) | 0.06 |
| 8 | Marketing Website | createEngine(), GeminiLiveEngine, addToRemoveQueue(), dispatch(), genId(), reducer(), toast() | 0.06 |
| 9 | Chat Bubble Component | handleKeyDown(), sendMessage(), assertRekognition(), authHeader(), blockAccess(), callMCP(), createBill(), createBillForUnit() (+24 more) | 0.10 |
| 10 | Admin AI Router | classifyAdminCategory(), routeAdminMessage(), selectAdminTool(), assignTicket(), getBuildingManager(), getLeastBusyStaff(), loadAssignmentRules(), CACHE_KEY() (+14 more) | 0.07 |
| 11 | Document Chunking (RAG) | buildLinePageMap(), chunkDocument(), createChunk(), detectHeading(), estimateTokens(), splitByParagraphs(), splitIntoSections(), splitWithOverlap() (+9 more) | 0.11 |
| 12 | Document Snap & Face Capture | computeEAR(), euclidean(), clearPhoto(), enroll(), ensureModels(), handleCapture(), handleFileSelect(), saveUploadEnrollment() (+2 more) | 0.09 |
| 13 | Building Systems & Engine | advanceStep(), CLAWBOT_STATE_KEY(), getClawBotState(), handleAccounting(), handleBuildingDetails(), handleBuildingStructure(), handleDocuments(), handleEmailForwarding() (+14 more) | 0.19 |
| 14 | AI Data Fetcher | fetchBaseContext(), fetchBuilding(), fetchIntentData(), fetchResident(), fetchResidentByPhone(), fetchUnit(), extractValidReferences(), validateAgentOutput() (+13 more) | 0.13 |
| 15 | Auth Context Builder | createSession(), destroySession(), getSession(), hydrateAliases(), parseRedisJson(), patchSession(), sessionKey(), userSessionKey() | 0.11 |
| 16 | Block Detection & Import | cleanName(), executeImport(), buildUnitLookupMap(), findExistingUnit(), normalizeUnitNumber(), formatPhoneForDB(), normalizePhone(), phoneMatchQuery() | 0.11 |
| 17 | Project Documentation Hub | Minilab Platform (Master Rulebook), Live Database Schema (165 Tables), Minilab Interface Contracts, AI Capability Audit (57 Handlers), Permissions and RLS Audit, 33 Tables with RLS but Zero Policies, Two-Stage AI Routing, DeepSeek V3 Routine AI Routing (85%) (+10 more) | 0.11 |
| 18 | Collection Gap Engine | buildTelegramMessage(), dispatchGapNotifications(), getBmTelegramIds(), tgSend(), resolveGap(), scanBuildingGaps(), scanComplianceExpiring(), scanFinanceCreditsLow() (+5 more) | 0.31 |
| 20 | Telegram Group Router | findGroupForCase(), notifyCaseToGroup(), botApi(), createInviteLink(), getGroupInfo(), getGroupMemberCount(), isUserInGroup(), sendGroupMessage() (+3 more) | 0.24 |
| 21 | Demo Data Management | clearAllDemoData(), clearDemoData(), deleteAllBuildingData(), getDemoIds(), pick(), randomDate(), randomMalaysianName(), randomPhone() (+4 more) | 0.36 |
| 22 | AWS Rekognition Provider | collectionId(), deleteFace(), ensureCollection(), getClient(), registerFace(), verifyFace() | 0.35 |
| 23 | File Scanner & Classifier | categorizeDocument(), storeDocumentMetadata(), buildReceiptData(), generateReceiptNumber() | 0.22 |
| 26 | Attendance RPCs & Queries | dayBoundsMYT(), getAttendanceOverview(), getAttendanceSparkline(), getAttendanceWeekly(), todayMYTBounds() | 0.36 |
| 27 | Telegram Group Service | addUsersToGroup(), createTelegramGroup(), deleteTelegramGroup(), getGroupInviteLink(), serviceCall() | 0.39 |
| 28 | Committee Page Handlers | handleAdd(), handleCommitteePosition(), handleEdit(), handleRemove(), loadStaff(), resetAddForm(), selectDepartment() | 0.29 |
| 30 | Global Search | getRecentSearches(), handleKeyDown(), navigateToResult(), saveRecentSearch() | 0.43 |
| 32 | Telegram Group Management | copyInviteLink(), deleteGroup(), friendlyError(), save(), sendInvites(), showToast() | 0.47 |
| 33 | Staff CRUD Handlers | clearForm(), clearPhoto(), handleDelete(), handleSave(), load(), resetForm() | 0.33 |
| 34 | Incident Analytics | getIncidentOverview(), getIncidentSparkline(), myttBoundsAgo() | 0.47 |
| 35 | File Parsing (CSV/Excel) | parseBuffer(), parseCSV(), parseFile(), parseXLSX(), splitCSVLine() | 0.67 |
| 36 | Load Testing | calculatePercentile(), generateTestMessage(), runLoadTest(), sendRequest(), simulateUser() | 0.60 |
| 37 | Invoice & Group Create | defaultGroupName(), fetchInvoices(), handleCreate(), resetCreateForm(), uploadFile() | 0.40 |
| 38 | Resident Import Wizard | callWizard(), handleImportResidents(), handleSectionA(), handleSectionB(), handleSectionD() | 0.40 |
| 41 | Sentry | captureError(), captureMessage(), sendToSentry() | 0.60 |
| 42 | Obo Provider | createOBOWaba(), fullOBOSetup(), registerPhoneNumber(), subscribeWebhooks() | 0.70 |
| 43 | Concept BM Console | BM Console Unit-Centric 3-Panel, sender_profiles Classification, Telethon Microservice on GCP, Quick Context (V20), V20 Session Handover | 0.40 |
| 44 | Page Confirmapplytemplate | confirmApplyTemplate(), flashSaved(), loadData(), togglePerm() | 0.67 |
| 45 | Page Closeslideover | closeSlideOver(), handleLinkContact(), handleSlideSubmit(), refreshDetails() | 0.50 |
| 50 | Page Formatdate | formatDate(), formatTime(), groupByDate() | 0.67 |
| 51 | Page Getdocstatus | getDocStatus(), isExpired(), isExpiringSoon() | 0.67 |
| 52 | Page Exportpdf | exportPdf(), fmt(), isImageUrl() | 0.67 |
| 53 | Page Handlerecordtxn | handleRecordTxn(), loadTransactions(), selectAccount() | 0.67 |
| 55 | Page Handledeleteresident | handleDeleteResident(), handleSaveResident(), showToastMsg() | 0.67 |
| 57 | Page Downloadblob | downloadBlob(), handleExport(), handlePdpaExport() | 0.67 |
| 67 | Concept Get Session | getSession() Auth (PWA Routes), requirePermission() Middleware, Staff Permissions Deep Audit | 0.67 |

## §Suggested-Questions
_Questions this graph is uniquely positioned to answer:_

- **What connects `Minilab Build Task List`, `V3 Build Requirements`, `V4 Build Requirements` to the rest of the system?**
  _30 weakly-connected nodes found - possible documentation gaps or missing edges._
- **Should `BM Portal Pages & UI` be split into smaller, more focused modules?**
  _Cohesion score 0.0 - nodes in this community are weakly interconnected._
- **Should `Shared UI Components` be split into smaller, more focused modules?**
  _Cohesion score 0.01 - nodes in this community are weakly interconnected._
- **Should `AI Pipeline & Sync` be split into smaller, more focused modules?**
  _Cohesion score 0.02 - nodes in this community are weakly interconnected._
- **Should `AI Providers & Tool Registry` be split into smaller, more focused modules?**
  _Cohesion score 0.02 - nodes in this community are weakly interconnected._
- **Should `Attendance & Data Import` be split into smaller, more focused modules?**
  _Cohesion score 0.03 - nodes in this community are weakly interconnected._
