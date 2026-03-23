---


---

<hr>
<h2 id="group-1-—-baseclearingaccountnumber-derivation">Group 1 — BaseClearingAccountNumber Derivation</h2>
<hr>
<p><strong>TC-LADS-01: Fidelity account number stripped to base [AUTO]</strong></p>
<p><strong>Requirement:</strong> REQ-1 — BaseClearingAccountNumber = ClearingAccountNumber minus last character (Fidelity)</p>
<p><strong>Preconditions:</strong></p>
<p>— <code>ClearingProvider = "Fidelity"</code> is configured</p>
<p>— ClearingAccountNumber = <code>ACF0012A</code></p>
<p><strong>Steps:</strong></p>
<ol>
<li>
<p>Invoke <code>FidelityBaseClearingAccountNumberConverter.Convert("ACF0012A")</code></p>
</li>
<li>
<p>Assert returned value</p>
</li>
</ol>
<p><strong>Expected Result:</strong></p>
<p>— Return value is <code>ACF0012</code> (last character <code>A</code> stripped)</p>
<p><strong>Negative checks:</strong></p>
<p>— Input <code>ACF0012</code> where <code>2</code> is the suffix → returns <code>ACF001</code> (digit suffix is valid and stripped; <code>ACF0012</code> = base <code>ACF001</code> + suffix <code>2</code>)</p>
<p>— Input empty string → returns <code>string.Empty</code> (<code>string.IsNullOrEmpty</code> guard fires, no exception thrown)</p>
<p>— Input null → returns <code>string.Empty</code> (same guard)</p>
<p>— Input single character <code>"A"</code> → returns <code>string.Empty</code> (<code>Length == 1</code> guard fires)</p>
<p><strong>Artifacts:</strong></p>
<pre><code>
Passed FidelityBaseClearingAccountNumberConverterTests.Convert_WithFidelityProvider_StripsLastCharacter("ACF0012A","ACF0012") [&lt; 1 ms]

Passed FidelityBaseClearingAccountNumberConverterTests.Convert_NullOrTooShort_ReturnsEmpty("") [&lt; 1 ms]

Passed FidelityBaseClearingAccountNumberConverterTests.Convert_NullOrTooShort_ReturnsEmpty(null) [&lt; 1 ms]

Passed FidelityBaseClearingAccountNumberConverterTests.Convert_NullOrTooShort_ReturnsEmpty("A") [&lt; 1 ms]

</code></pre>
<hr>
<p><strong>TC-LADS-02: Multi-character suffix accounts [AUTO]</strong></p>
<p><strong>Requirement:</strong> REQ-1 — BaseClearingAccountNumber derivation is suffix-agnostic (strips exactly one last character)</p>
<p><strong>Preconditions:</strong></p>
<p>— <code>ClearingProvider = "Fidelity"</code></p>
<p><strong>Steps:</strong></p>
<ol>
<li>
<p>Invoke <code>FidelityBaseClearingAccountNumberConverter.Convert("ACF5678E")</code> → assert <code>ACF5678</code></p>
</li>
<li>
<p>Invoke <code>FidelityBaseClearingAccountNumberConverter.Convert("ACF5678F")</code> → assert <code>ACF5678</code></p>
</li>
<li>
<p>Invoke <code>FidelityBaseClearingAccountNumberConverter.Convert("ACF5678M")</code> → assert <code>ACF5678</code></p>
</li>
</ol>
<p><strong>Expected Result:</strong></p>
<p>— All three inputs produce <code>ACF5678</code></p>
<p><strong>Negative checks:</strong></p>
<p>— Numeric suffix <code>ACF56781</code> → returns <code>ACF5678</code> (numeric last char also stripped)</p>
<p><strong>Artifacts:</strong></p>
<pre><code>
Passed FidelityBaseClearingAccountNumberConverterTests.Convert_WithFidelityProvider_StripsLastCharacter("ACF5678E","ACF5678") [&lt; 1 ms]

Passed FidelityBaseClearingAccountNumberConverterTests.Convert_WithFidelityProvider_StripsLastCharacter("ACF5678F","ACF5678") [&lt; 1 ms]

Passed FidelityBaseClearingAccountNumberConverterTests.Convert_WithFidelityProvider_StripsLastCharacter("ACF5678M","ACF5678") [&lt; 1 ms]

Passed FidelityBaseClearingAccountNumberConverterTests.Convert_WithFidelityProvider_StripsLastCharacter("ACF56781","ACF5678") [&lt; 1 ms]

</code></pre>
<hr>
<h2 id="group-2-—-db-record-creation-per-sub-account">Group 2 — DB Record Creation per Sub-Account</h2>
<hr>
<p><strong>TC-LADS-03: Lambda creates one DB row per sub-account for a new document [AUTO]</strong></p>
<p><strong>Requirement:</strong> REQ-1, REQ-4 — Document must be associated with ALL sub-accounts</p>
<blockquote>
<p><strong>[NOTE]</strong> PR 15405 creates exactly <strong>1 DB row per 1 SFTP file processed</strong>. For 2 rows to exist, SFTP must deliver <strong>2 separate files</strong> (one per sub-account: <code>ACF1234E</code>, <code>ACF1234F</code>). If instead the feature works by delivering 1 base-account file and expanding to sub-accounts via AMS DB lookup, that expansion logic must be in PR 15404 — confirm the SFTP delivery pattern and PR 15404 scope before finalizing this TC.</p>
</blockquote>
<p><strong>Preconditions:</strong></p>
<p>— <code>S3AccountDocumentInfos</code> table is empty for this base account</p>
<p>— SFTP path contains <strong>one AccountStatement file per sub-account</strong>: <code>ACF1234E</code> and <code>ACF1234F</code></p>
<p>— Lambda payload: <code>ClearingProvider="Fidelity"</code>, correct <code>AccountStatementOptions</code> path template</p>
<p><strong>Steps:</strong></p>
<ol>
<li>
<p>Invoke Lambda (or <code>FileOperationService.ProcessSingleFileAsync</code> under test) for both files</p>
</li>
<li>
<p>Query <code>S3AccountDocumentInfos</code> WHERE <code>BaseClearingAccountNumber = 'ACF1234'</code></p>
</li>
<li>
<p>Count rows</p>
</li>
</ol>
<p><strong>Expected Result:</strong></p>
<p>— Exactly <strong>2 rows</strong> exist in <code>S3AccountDocumentInfos</code> (one per processed sub-account file)</p>
<p>— Row 1: <code>ClearingAccountNumber = 'ACF1234E'</code>, <code>BaseClearingAccountNumber = 'ACF1234'</code></p>
<p>— Row 2: <code>ClearingAccountNumber = 'ACF1234F'</code>, <code>BaseClearingAccountNumber = 'ACF1234'</code></p>
<p><strong>Negative checks:</strong></p>
<p>— No third row exists with any other <code>ClearingAccountNumber</code></p>
<p><strong>Artifacts:</strong></p>
<pre class=" language-sql"><code class="prism  language-sql">
<span class="token keyword">SELECT</span> ClearingAccountNumber<span class="token punctuation">,</span> BaseClearingAccountNumber<span class="token punctuation">,</span> Path<span class="token punctuation">,</span> <span class="token keyword">Type</span>

<span class="token keyword">FROM</span> S3AccountDocumentInfos

<span class="token keyword">WHERE</span> BaseClearingAccountNumber <span class="token operator">=</span> <span class="token string">'ACF1234'</span>

  

<span class="token comment">-- Expected output:</span>

ClearingAccountNumber <span class="token operator">|</span> BaseClearingAccountNumber <span class="token operator">|</span> Path <span class="token operator">|</span> <span class="token keyword">Type</span>

ACF1234E <span class="token operator">|</span> ACF1234 <span class="token operator">|</span> test<span class="token operator">-</span>env<span class="token operator">/</span>AccountStatement<span class="token operator">/</span>ACF1234E<span class="token operator">/</span><span class="token number">2026</span><span class="token operator">/</span><span class="token number">03</span><span class="token operator">/</span><span class="token number">23</span><span class="token operator">/</span>statement<span class="token punctuation">.</span>pdf <span class="token operator">|</span> AccountStatement

ACF1234F <span class="token operator">|</span> ACF1234 <span class="token operator">|</span> test<span class="token operator">-</span>env<span class="token operator">/</span>AccountStatement<span class="token operator">/</span>ACF1234F<span class="token operator">/</span><span class="token number">2026</span><span class="token operator">/</span><span class="token number">03</span><span class="token operator">/</span><span class="token number">23</span><span class="token operator">/</span>statement<span class="token punctuation">.</span>pdf <span class="token operator">|</span> AccountStatement

<span class="token punctuation">(</span><span class="token number">2</span>  <span class="token keyword">rows</span><span class="token punctuation">)</span>

</code></pre>
<hr>
<p><strong>TC-LADS-04: Three sub-accounts all receive DB records [AUTO]</strong></p>
<p><strong>Requirement:</strong> REQ-1 — Applies to accounts with 3+ sub-accounts</p>
<blockquote>
<p><strong>[NOTE]</strong> Same architectural precondition as TC-LADS-03 — SFTP delivers 3 separate files.</p>
</blockquote>
<p><strong>Preconditions:</strong></p>
<p>— No existing records for this base account</p>
<p>— SFTP has files for <code>ACF5678E</code>, <code>ACF5678F</code>, <code>ACF5678M</code></p>
<p><strong>Steps:</strong></p>
<ol>
<li>
<p>Invoke Lambda for all three files</p>
</li>
<li>
<p>Query <code>S3AccountDocumentInfos</code> WHERE <code>BaseClearingAccountNumber = 'ACF5678'</code></p>
</li>
<li>
<p>Count rows and verify <code>ClearingAccountNumber</code> values</p>
</li>
</ol>
<p><strong>Expected Result:</strong></p>
<p>— Exactly <strong>3 rows</strong> returned</p>
<p>— <code>ClearingAccountNumber</code> values are <code>{ACF5678E, ACF5678F, ACF5678M}</code> (all three present, no duplicates)</p>
<p><strong>Negative checks:</strong></p>
<p>— No row with <code>ClearingAccountNumber = 'ACF5678'</code> (bare base account does not appear on SFTP for Fidelity)</p>
<p><strong>Artifacts:</strong></p>
<pre class=" language-sql"><code class="prism  language-sql">
<span class="token keyword">SELECT</span> ClearingAccountNumber<span class="token punctuation">,</span> BaseClearingAccountNumber

<span class="token keyword">FROM</span> S3AccountDocumentInfos

<span class="token keyword">WHERE</span> BaseClearingAccountNumber <span class="token operator">=</span> <span class="token string">'ACF5678'</span>

  

<span class="token comment">-- Expected output:</span>

ClearingAccountNumber <span class="token operator">|</span> BaseClearingAccountNumber

ACF5678E <span class="token operator">|</span> ACF5678

ACF5678F <span class="token operator">|</span> ACF5678

ACF5678M <span class="token operator">|</span> ACF5678

<span class="token punctuation">(</span><span class="token number">3</span>  <span class="token keyword">rows</span><span class="token punctuation">)</span>

</code></pre>
<hr>
<p><strong>TC-LADS-05: All document types receive sub-account rows (AccountStatement, TaxForm, TradeConfirmation) [AUTO]</strong></p>
<p><strong>Requirement:</strong> REQ-4 — Feature applies to all three document types</p>
<blockquote>
<p><strong>[NOTE]</strong> Same architectural precondition — 2 SFTP files per document type (one per sub-account).</p>
</blockquote>
<p><strong>Preconditions:</strong></p>
<p>— SFTP has one file of each type for <code>ACF1234E</code> and <code>ACF1234F</code> (6 files total)</p>
<p><strong>Steps:</strong></p>
<ol>
<li>
<p>Invoke Lambda with all three document type options configured</p>
</li>
<li>
<p>Query <code>S3AccountDocumentInfos</code> WHERE <code>BaseClearingAccountNumber = 'ACF1234'</code> grouped by <code>Type</code></p>
</li>
</ol>
<p><strong>Expected Result:</strong></p>
<p>— <code>Type = AccountStatement</code>: 2 rows</p>
<p>— <code>Type = TaxForm</code>: 2 rows</p>
<p>— <code>Type = TradeConfirmation</code>: 2 rows</p>
<p>— Total: 6 rows</p>
<p><strong>Negative checks:</strong></p>
<p>— No row has <code>Type</code> value outside <code>{AccountStatement, TaxForm, TradeConfirmation}</code></p>
<p><strong>Artifacts:</strong></p>
<pre class=" language-sql"><code class="prism  language-sql">
<span class="token keyword">SELECT</span>  <span class="token keyword">Type</span><span class="token punctuation">,</span> <span class="token function">COUNT</span><span class="token punctuation">(</span><span class="token operator">*</span><span class="token punctuation">)</span> <span class="token keyword">AS</span> <span class="token keyword">RowCount</span>

<span class="token keyword">FROM</span> S3AccountDocumentInfos

<span class="token keyword">WHERE</span> BaseClearingAccountNumber <span class="token operator">=</span> <span class="token string">'ACF1234'</span>

<span class="token keyword">GROUP</span> <span class="token keyword">BY</span>  <span class="token keyword">Type</span>

  

<span class="token comment">-- Expected output:</span>

<span class="token keyword">Type</span> <span class="token operator">|</span> <span class="token keyword">RowCount</span>

AccountStatement <span class="token operator">|</span> <span class="token number">2</span>

TaxForm <span class="token operator">|</span> <span class="token number">2</span>

TradeConfirmation <span class="token operator">|</span> <span class="token number">2</span>

<span class="token punctuation">(</span><span class="token number">3</span>  <span class="token keyword">rows</span><span class="token punctuation">,</span> total <span class="token number">6</span><span class="token punctuation">)</span>

</code></pre>
<hr>
<h2 id="group-3-—-s3-deduplication">Group 3 — S3 Deduplication</h2>
<hr>
<p><strong>TC-LADS-06: S3 object path uses full sub-account number (with suffix) [AUTO]</strong></p>
<p><strong>Requirement:</strong> REQ-3 — S3 path confirmed from <code>FileOperationService.BuildS3Key</code></p>
<p><strong>Preconditions:</strong></p>
<p>— SFTP has a file for <code>ACF1234E</code></p>
<p>— S3 bucket is empty</p>
<p><strong>Steps:</strong></p>
<ol>
<li>
<p>Invoke Lambda for one AccountStatement file for <code>ACF1234E</code></p>
</li>
<li>
<p>List S3 objects in <code>{DestinationBucket}</code></p>
</li>
</ol>
<p><strong>Expected Result:</strong></p>
<p>— Exactly <strong>1</strong> S3 object exists</p>
<p>— S3 path follows the structure (confirmed from <code>BuildS3Key</code>):</p>
<p><code>{EnvironmentName}/{DocumentType}/{ClearingAccountNumber}/{YYYY}/{MM}/{DD}/{filename}</code></p>
<p>Example: <code>prod/AccountStatement/ACF1234E/2026/03/23/statement.pdf</code></p>
<p>— <code>BaseClearingAccountNumber</code> (<code>ACF1234</code>) does <strong>not</strong> appear in the S3 path</p>
<p><strong>Negative checks:</strong></p>
<p>— No S3 object exists under a base account path (<code>ACF1234/</code>) — S3 is keyed by full sub-account number</p>
<p><strong>Artifacts:</strong></p>
<pre><code>
$ aws s3 ls s3://test-bucket/test-env/AccountStatement/ACF1234E/2026/03/23/

2026-03-23 10:00:00 123456 statement.pdf

(1 object, 0 objects under ACF1234/)

</code></pre>
<hr>
<p><strong>TC-LADS-07: Lambda does not re-upload file when S3 object and DB row both exist [AUTO]</strong></p>
<p><strong>Requirement:</strong> Deduplication — <code>ShouldSkipExistingFile</code>: both S3 + DB metadata present → skip</p>
<p><strong>Preconditions:</strong></p>
<p>— S3 already contains the file at <code>{env}/AccountStatement/ACF1234E/{date}/statement.pdf</code></p>
<p>— <code>S3AccountDocumentInfos</code> has a row for <code>ACF1234E</code> with matching <code>Path</code> and <code>BucketName</code></p>
<p><strong>Steps:</strong></p>
<ol>
<li>
<p>Record S3 object count and DB row count before Lambda run</p>
</li>
<li>
<p>Invoke Lambda</p>
</li>
<li>
<p>Assert S3 object count unchanged</p>
</li>
<li>
<p>Assert DB row count unchanged</p>
</li>
<li>
<p>Check CloudWatch logs</p>
</li>
</ol>
<p><strong>Expected Result:</strong></p>
<p>— S3 object count: unchanged</p>
<p>— DB row count: unchanged</p>
<p>— Log contains <code>"File already exists in S3: {bucket}/{key}, skipping upload"</code></p>
<p><strong>Negative checks:</strong></p>
<p>— No new rows inserted into <code>S3AccountDocumentInfos</code></p>
<p><strong>Artifacts:</strong></p>
<pre><code>
-- DB row count before: 1, after: 1 (unchanged)

-- S3 object count before: 1, after: 1 (unchanged)

  

[INFO] File already exists in S3: test-bucket/test-env/AccountStatement/ACF1234E/2026/03/23/statement.pdf, skipping upload

</code></pre>
<hr>
<h2 id="group-4-—-retroactive-backfill">Group 4 — Retroactive Backfill</h2>
<hr>
<p><strong>TC-LADS-08: File exists in S3 but DB row missing — Lambda recreates metadata without re-uploading [AUTO]</strong></p>
<p><strong>Requirement:</strong> REQ-5 — <code>TryAddMissingMetadataToDb</code>: S3 object present, DB row absent → add row, skip upload</p>
<p><strong>Preconditions:</strong></p>
<p>— S3 already contains file at <code>{env}/AccountStatement/ACF1234E/{date}/statement.pdf</code></p>
<p>— <code>S3AccountDocumentInfos</code> has <strong>no row</strong> matching that <code>BucketName</code> + <code>Path</code></p>
<p><strong>Steps:</strong></p>
<ol>
<li>
<p>Record DB row count (expect: 0 for this file path)</p>
</li>
<li>
<p>Invoke Lambda for <code>ACF1234E</code></p>
</li>
<li>
<p>Query <code>S3AccountDocumentInfos</code> WHERE <code>ClearingAccountNumber = 'ACF1234E'</code></p>
</li>
</ol>
<p><strong>Expected Result:</strong></p>
<p>— <strong>1 row</strong> created: <code>ClearingAccountNumber = 'ACF1234E'</code>, <code>BaseClearingAccountNumber = 'ACF1234'</code></p>
<p>— No new S3 upload (file already in S3)</p>
<p>— CloudWatch log shows <code>"File exists in S3 but metadata is missing. Adding metadata to DB"</code></p>
<p><strong>Negative checks:</strong></p>
<p>— S3 object count unchanged (no re-upload)</p>
<p><strong>Artifacts:</strong></p>
<pre><code>
-- DB row count before: 0, after: 1

  

[INFO] File exists in S3 but metadata is missing. Adding metadata to DB: test-bucket/test-env/AccountStatement/ACF1234E/2026/03/23/statement.pdf

[INFO] Successfully added metadata to DB for: test-env/AccountStatement/ACF1234E/2026/03/23/statement.pdf

  

SELECT Id, ClearingAccountNumber, BaseClearingAccountNumber FROM S3AccountDocumentInfos WHERE ClearingAccountNumber = 'ACF1234E'

Id | ClearingAccountNumber | BaseClearingAccountNumber

xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx | ACF1234E | ACF1234

(1 row)

</code></pre>
<hr>
<p><strong>TC-LADS-09: Idempotency — running Lambda twice does not duplicate rows [AUTO]</strong></p>
<p><strong>Requirement:</strong> REQ-3 / REQ-5 — No duplicate records on repeated runs</p>
<p><strong>Preconditions:</strong></p>
<p>— SFTP has files for <code>ACF1234E</code> and <code>ACF1234F</code></p>
<p><strong>Steps:</strong></p>
<ol>
<li>
<p>Invoke Lambda (first run)</p>
</li>
<li>
<p>Record DB row count and all row <code>Id</code> values</p>
</li>
<li>
<p>Invoke Lambda (second run)</p>
</li>
<li>
<p>Re-query DB</p>
</li>
</ol>
<p><strong>Expected Result:</strong></p>
<p>— Row count after second run = row count after first run</p>
<p>— No new rows inserted on second run</p>
<p>— CloudWatch logs for second run show <code>"File already exists in S3: {bucket}/{key}, skipping upload"</code></p>
<p><strong>Negative checks:</strong></p>
<p>— Zero rows with duplicate <code>(Path, ClearingAccountNumber)</code> combination</p>
<p><strong>Artifacts:</strong></p>
<pre><code>
-- Run 1: row count = 2, Ids: [aaa..., bbb...]

-- Run 2: row count = 2, Ids: [aaa..., bbb...] (same — no new rows)

  

[INFO] File already exists in S3: test-bucket/test-env/AccountStatement/ACF1234E/2026/03/23/statement.pdf, skipping upload

[INFO] File already exists in S3: test-bucket/test-env/AccountStatement/ACF1234F/2026/03/23/statement.pdf, skipping upload

  

SELECT COUNT(*), ClearingAccountNumber FROM S3AccountDocumentInfos

WHERE BaseClearingAccountNumber = 'ACF1234'

GROUP BY ClearingAccountNumber

-- Expected: ACF1234E → 1, ACF1234F → 1 (no duplicates)

</code></pre>
<hr>
<h2 id="group-5-—-widget-visibility">Group 5 — Widget Visibility</h2>
<hr>
<p><strong>TC-LADS-10: Document appears in widget for ACF1234E account [MANUAL]</strong></p>
<p><strong>Requirement:</strong> REQ-2 — Document visible in widget for each sub-account</p>
<p><strong>Preconditions:</strong></p>
<p>— Lambda has run and created DB rows for ACF1234E and ACF1234F</p>
<p>— User account bound to ACF1234E in the platform</p>
<p><strong>Steps:</strong></p>
<ol>
<li>
<p>Login to the platform</p>
</li>
<li>
<p>Bind account ACF1234E to the logged-in user</p>
</li>
<li>
<p>Select account ACF1234E and add the Account Documents widget</p>
</li>
<li>
<p>Filter by document type “Account Statement” and relevant date</p>
</li>
</ol>
<p><strong>Expected Result:</strong></p>
<p>— At least one AccountStatement document appears in the widget for ACF1234E</p>
<p>— Document is downloadable (clicking download initiates PDF download without error)</p>
<p><strong>Negative checks:</strong></p>
<p>— Widget does not show an error or empty state when documents exist in DB</p>
<p><strong>Artifacts:</strong></p>
<p>— Screenshot of widget with document visible, browser console log</p>
<hr>
<p><strong>TC-LADS-11: Document appears in widget for ACF1234F account [MANUAL]</strong></p>
<p><strong>Requirement:</strong> REQ-2 — Same document visible for the other sub-account</p>
<p><strong>Preconditions:</strong></p>
<p>— Same as TC-LADS-10; user bound to ACF1234F instead</p>
<p><strong>Steps:</strong></p>
<ol>
<li>
<p>Login to the platform</p>
</li>
<li>
<p>Bind account ACF1234F to the logged-in user</p>
</li>
<li>
<p>Select account ACF1234F and add the Account Documents widget</p>
</li>
<li>
<p>Filter by document type and date matching TC-LADS-10</p>
</li>
</ol>
<p><strong>Expected Result:</strong></p>
<p>— The <strong>same document</strong> (same filename / date) appears in ACF1234F widget as seen in ACF1234E widget</p>
<p>— Document is downloadable</p>
<p><strong>Negative checks:</strong></p>
<p>— Widget does not display documents intended for a different base account</p>
<p><strong>Artifacts:</strong></p>
<p>— Screenshot, browser network HAR showing widget API request with <code>BaseClearingAccountNumber=ACF1234</code></p>
<hr>
<p><strong>TC-LADS-12: Widget shows documents for all three sub-accounts (ACF5678E, F, M) [MANUAL]</strong></p>
<p><strong>Requirement:</strong> REQ-2 — Applies to accounts with 3+ sub-accounts</p>
<p><strong>Preconditions:</strong></p>
<p>— Lambda run for base account ACF5678 created 3 DB rows (one per sub-account file)</p>
<p><strong>Steps:</strong></p>
<ol>
<li>
<p>Repeat TC-LADS-10/11 procedure for each of ACF5678E, ACF5678F, ACF5678M</p>
</li>
<li>
<p>Note document count and document names in each sub-account widget</p>
</li>
</ol>
<p><strong>Expected Result:</strong></p>
<p>— All three sub-account widgets show the same document</p>
<p><strong>Negative checks:</strong></p>
<p>— No sub-account shows 0 documents when the other two show documents</p>
<p><strong>Artifacts:</strong></p>
<p>— Screenshots for each sub-account widget</p>
<hr>
<h2 id="group-6-—-negative--error-cases">Group 6 — Negative / Error Cases</h2>
<hr>
<p><strong>TC-LADS-13: Every Fidelity account number always has a suffix — derivation is always valid [AUTO]</strong></p>
<p><strong>Requirement:</strong> REQ-1 — Confirmed design invariant: for Fidelity all account numbers contain a suffix character. <code>ACF1234</code> = base <code>ACF123</code> + suffix <code>4</code>. There are no bare/suffix-less accounts. Strip-last-char is always safe.</p>
<p><strong>Preconditions:</strong></p>
<p>— <code>ClearingProvider = "Fidelity"</code></p>
<p>— Account <code>ACF1234</code> (suffix = <code>4</code>, base = <code>ACF123</code>)</p>
<p><strong>Steps:</strong></p>
<ol>
<li>
<p>Invoke <code>FidelityBaseClearingAccountNumberConverter.Convert("ACF1234")</code></p>
</li>
<li>
<p>Assert result</p>
</li>
</ol>
<p><strong>Expected Result:</strong></p>
<p>— Returns <code>ACF123</code> (last character <code>4</code> stripped — a digit suffix is valid)</p>
<p><strong>Negative checks:</strong></p>
<p>— No scenario where Fidelity delivers a truly suffix-less account exists; <code>length == 1</code> guard (<code>string.Empty</code>) covers pathological edge case only</p>
<p><strong>Artifacts:</strong></p>
<pre><code>
Passed FidelityBaseClearingAccountNumberConverterTests.Convert_WithFidelityProvider_StripsLastCharacter("ACF1234","ACF123") [&lt; 1 ms]

</code></pre>
<hr>
<p><strong>TC-LADS-14: <code>BaseClearingAccountNumber</code> column missing from DB — Lambda fails gracefully [MANUAL]</strong></p>
<p><strong>Requirement:</strong> Resilience — DB schema validation</p>
<p><strong>Preconditions:</strong></p>
<p>— AMS DB <code>S3AccountDocumentInfos</code> table does NOT have <code>BaseClearingAccountNumber</code> column (simulating missing migration)</p>
<p><strong>Steps:</strong></p>
<ol>
<li>
<p>Invoke Lambda</p>
</li>
<li>
<p>Check Lambda output status and CloudWatch logs</p>
</li>
</ol>
<p><strong>Expected Result:</strong></p>
<p>— Lambda returns <code>Failed</code> status</p>
<p>— CloudWatch log contains an error indicating the column/schema issue</p>
<p>— No partial data written to S3 or DB</p>
<p><strong>Negative checks:</strong></p>
<p>— Lambda does not silently succeed with 0 documents processed</p>
<p><strong>Artifacts:</strong></p>
<p>— CloudWatch log error, Lambda return status</p>
<hr>
<p><strong>TC-LADS-15: Non-Fidelity provider — BaseClearingAccountNumber equals ClearingAccountNumber [AUTO]</strong></p>
<p><strong>Requirement:</strong> Out-of-scope confirmation — only Fidelity accounts affected by new derivation logic</p>
<p><strong>Preconditions:</strong></p>
<p>— <code>ClearingProvider</code> set to any non-Fidelity value (e.g., <code>Apex</code>, <code>InteliClear</code>, <code>None</code>)</p>
<p>— Account number <code>XYZ1234A</code></p>
<p><strong>Steps:</strong></p>
<ol>
<li>
<p>Invoke <code>DefaultBaseClearingAccountNumberConverter.Convert("XYZ1234A")</code></p>
</li>
<li>
<p>Assert <code>BaseClearingAccountNumber</code></p>
</li>
</ol>
<p><strong>Expected Result:</strong></p>
<p>— <code>BaseClearingAccountNumber = "XYZ1234A"</code> (identity — full account number returned as-is)</p>
<p>— Confirmed from <code>DefaultBaseClearingAccountNumberConverter</code>: <code>return clearingAccountNumber ?? string.Empty</code></p>
<p><strong>Negative checks:</strong></p>
<p>— Input null → returns <code>string.Empty</code> (no exception)</p>
<p><strong>Artifacts:</strong></p>
<pre><code>
Passed DefaultBaseClearingAccountNumberConverterTests.Convert_NonFidelityProvider_ReturnsInputUnchanged("XYZ1234A","XYZ1234A") [&lt; 1 ms]

Passed DefaultBaseClearingAccountNumberConverterTests.Convert_Null_ReturnsEmpty [&lt; 1 ms]

Passed BaseClearingAccountNumberConverterFactoryTests.GetConverter_NonFidelity_ReturnsDefaultConverter(None) [&lt; 1 ms]

Passed BaseClearingAccountNumberConverterFactoryTests.GetConverter_NonFidelity_ReturnsDefaultConverter(Apex) [&lt; 1 ms]

</code></pre>
<hr>
<p><strong>TC-LADS-16: Orphaned S3 metadata cleanup — file missing in S3 but DB row exists [AUTO]</strong></p>
<p><strong>Requirement:</strong> Existing behavior — <code>TryDeleteOrphanedMetadataFromDb</code>: DB row present, S3 object absent → delete row, then re-upload</p>
<p><strong>Preconditions:</strong></p>
<p>— <code>S3AccountDocumentInfos</code> has a row for <code>ACF1234E</code> with <code>Path = '{env}/AccountStatement/ACF1234E/{date}/statement.pdf'</code></p>
<p>— The S3 object at that path has been manually deleted</p>
<p><strong>Steps:</strong></p>
<ol>
<li>
<p>Invoke Lambda for <code>ACF1234E</code></p>
</li>
<li>
<p>Check <code>S3AccountDocumentInfos</code> for <code>ACF1234E</code> row after run</p>
</li>
<li>
<p>Check CloudWatch logs</p>
</li>
</ol>
<p><strong>Expected Result:</strong></p>
<p>— CloudWatch log contains <code>"Metadata exists in DB but file is missing in S3. Deleting orphaned metadata: {bucket}/{key}"</code></p>
<p>— Orphaned DB row is removed</p>
<p>— Lambda re-uploads the file and creates a new DB row for <code>ACF1234E</code></p>
<p><strong>Negative checks:</strong></p>
<p>— Row count after run = 1 (orphaned row removed, new row created — not 2 rows)</p>
<p><strong>Artifacts:</strong></p>
<pre><code>
-- DB row count before: 1 (stale row with Id=aaa...)

-- DB row count after: 1 (new row with Id=bbb... — different Id)

  

[INFO] Metadata exists in DB but file is missing in S3. Deleting orphaned metadata: test-bucket/test-env/AccountStatement/ACF1234E/2026/03/23/statement.pdf

[INFO] Successfully deleted orphaned metadata from DB for: test-env/AccountStatement/ACF1234E/2026/03/23/statement.pdf

[INFO] Uploading to S3: test-bucket/test-env/AccountStatement/ACF1234E/2026/03/23/statement.pdf

[INFO] Successfully uploaded file to S3: test-env/AccountStatement/ACF1234E/2026/03/23/statement.pdf

</code></pre>
<hr>
<p><strong>TC-LADS-17: Lambda payload missing <code>ClearingProvider</code> — defaults to identity derivation [AUTO]</strong></p>
<p><strong>Requirement:</strong> Defensive — <code>ClearingProvider</code> is optional; defaults to <code>None = 0</code> → <code>DefaultBaseClearingAccountNumberConverter</code></p>
<p><strong>Preconditions:</strong></p>
<p>— Lambda invoked WITHOUT <code>ClearingProvider</code> field in payload</p>
<p><strong>Steps:</strong></p>
<ol>
<li>
<p>Invoke Lambda without <code>ClearingProvider</code></p>
</li>
<li>
<p>Check Lambda status, CloudWatch logs, and resulting DB row</p>
</li>
</ol>
<p><strong>Expected Result:</strong></p>
<p>— Lambda runs without exception (<code>ClearingProvider</code> enum defaults to <code>None = 0</code>)</p>
<p>— <code>DefaultBaseClearingAccountNumberConverter</code> is used</p>
<p>— <code>BaseClearingAccountNumber = ClearingAccountNumber</code> (full account number, no stripping)</p>
<p>— DB row created with <code>BaseClearingAccountNumber</code> equal to full <code>ClearingAccountNumber</code></p>
<p><strong>Negative checks:</strong></p>
<p>— No <code>NullReferenceException</code> or unhandled exception in CloudWatch</p>
<p><strong>Artifacts:</strong></p>
<pre><code>
-- Lambda status: Succeeded

  

SELECT ClearingAccountNumber, BaseClearingAccountNumber FROM S3AccountDocumentInfos WHERE ClearingAccountNumber = 'ACF7777E'

ClearingAccountNumber | BaseClearingAccountNumber

ACF7777E | ACF7777E -- BaseClearingAccountNumber = ClearingAccountNumber (no stripping)

(1 row)

</code></pre>
<hr>
<p><strong>TC-LADS-18: Row count reflects actual SFTP files — no hardcoded suffix expansion [AUTO]</strong></p>
<p><strong>Requirement:</strong> REQ-1 — Sub-account rows driven by SFTP content, not hardcoded suffix list</p>
<blockquote>
<p><strong>[NOTE]</strong> Validates that the Lambda does NOT generate rows for sub-accounts not present on SFTP (i.e., no artificial expansion beyond what SFTP delivers).</p>
</blockquote>
<p><strong>Preconditions:</strong></p>
<p>— SFTP has a file for <code>ACF7777E</code> only (not <code>ACF7777F</code>)</p>
<p><strong>Steps:</strong></p>
<ol>
<li>
<p>Invoke Lambda for <code>ACF7777E</code></p>
</li>
<li>
<p>Query <code>S3AccountDocumentInfos</code> WHERE <code>BaseClearingAccountNumber = 'ACF7777'</code></p>
</li>
</ol>
<p><strong>Expected Result:</strong></p>
<p>— Exactly <strong>1 row</strong>, <code>ClearingAccountNumber = 'ACF7777E'</code></p>
<p><strong>Negative checks:</strong></p>
<p>— No row with <code>ClearingAccountNumber = 'ACF7777F'</code> (not on SFTP → not created)</p>
<p><strong>Artifacts:</strong></p>
<pre class=" language-sql"><code class="prism  language-sql">
<span class="token keyword">SELECT</span> ClearingAccountNumber<span class="token punctuation">,</span> BaseClearingAccountNumber

<span class="token keyword">FROM</span> S3AccountDocumentInfos

<span class="token keyword">WHERE</span> BaseClearingAccountNumber <span class="token operator">=</span> <span class="token string">'ACF7777'</span>

  

<span class="token comment">-- Expected output:</span>

ClearingAccountNumber <span class="token operator">|</span> BaseClearingAccountNumber

ACF7777E <span class="token operator">|</span> ACF7777

<span class="token punctuation">(</span><span class="token number">1</span>  <span class="token keyword">row</span><span class="token punctuation">)</span> <span class="token comment">-- no row for ACF7777F</span>

</code></pre>
<hr>

