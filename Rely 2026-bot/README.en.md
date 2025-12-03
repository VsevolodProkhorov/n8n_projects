<h1>RELY2026 Telegram Product Parser Bot (n8n Workflow)</h1>

<p>
  <strong>RELY2026 Bot</strong> is an automation workflow built in <strong>n8n</strong> that processes
  Telegram channel posts, extracts product data from captions, matches it with photos from album posts,
  and uploads fully structured product records into <strong>Airtable</strong>.
</p>

<p>
  The goal of this workflow is to <strong>fully automate the creation of a product catalog</strong>
  based on Telegram posts — without any manual data entry.
</p>

<hr />

<h2>📌 Key Features</h2>

<ul>
  <li>
    <strong>Automatic message ingestion from a Telegram channel</strong>  
    using the <code>Telegram Trigger</code>.
  </li>

  <li>
    <strong>Full Telegram album (media_group) support</strong>:
    <ul>
      <li>each photo in an album is buffered in a DataTable,</li>
      <li>the workflow waits until the final photo with a caption arrives,</li>
      <li>once the album is complete, the full batch is processed.</li>
    </ul>
  </li>

  <li>
    <strong>Advanced Python caption parser</strong> that:
    <ul>
      <li>detects individual products by detecting 5-digit product numbers,</li>
      <li>extracts brand names (via intelligent English-word detection),</li>
      <li>detects colours, fabric types, and material keywords,</li>
      <li>parses EU and letter sizes (including ranges like <code>38-40</code> or <code>S–XL</code>),</li>
      <li>extracts <strong>new</strong> and <strong>old</strong> prices,</li>
      <li>creates <strong>separate product rows per size</strong>.</li>
    </ul>
  </li>

  <li>
    <strong>Image-to-item matching</strong>:
    <ul>
      <li>Python assigns incremental <code>id_fotos</code> to photos,</li>
      <li>the parser assigns <code>id_item</code> to each product entry,</li>
      <li>the <strong>Merge</strong> node aligns products and photos 1:1.</li>
    </ul>
  </li>

  <li>
    <strong>Uploads structured product data into Airtable</strong>:
    <ul>
      <li>Product Photo (direct Telegram URL)</li>
      <li>product_number</li>
      <li>Brand</li>
      <li>Size</li>
      <li>Price</li>
      <li>Colour</li>
      <li>Description</li>
    </ul>
  </li>

  <li><strong>Clears buffer DataTable rows</strong> to prevent duplicate processing.</li>

  <li><strong>Sends a Telegram confirmation message</strong>:
    <em>"Data successfully added to the database!"</em>
  </li>
</ul>

<hr />

<h2>🧠 Workflow Logic Breakdown</h2>

<ol>
  <li>
    <h3>Telegram message received</h3>
    <p>The <code>Telegram Trigger</code> receives a new channel message.</p>
    <p>
      The first <code>If</code> node checks:
    </p>
    <ul>
      <li><strong>If it’s part of an album</strong> (<code>media_group_id</code> exists) → photo is buffered.</li>
      <li><strong>If it’s an unrelated message</strong> → ignored.</li>
    </ul>
  </li>

  <li>
    <h3>Saving photo metadata into DataTable</h3>
    <p>The <strong>Saving</strong> node stores:</p>
    <ul>
      <li><code>media_group_id</code></li>
      <li><code>chat_id</code></li>
      <li><code>message_id</code></li>
      <li><code>file_id</code></li>
      <li><code>caption</code> (only present in the last message of an album)</li>
    </ul>
  </li>

  <li>
    <h3>Detecting album completion</h3>
    <p>
      The second <code>If</code> node checks whether <code>caption</code> exists.  
      Telegram only includes <strong>one caption per album</strong> — on the last photo.
    </p>
    <p>This means: <strong>the album is complete and ready for processing.</strong></p>
  </li>

  <li>
    <h3>Wait for Telegram metadata synchronization</h3>
    <p>
      The <code>Wait1</code> node delays processing for 5 seconds to ensure  
      Telegram finishes posting all images and metadata.
    </p>
  </li>

  <li>
    <h3>Retrieving album data from DataTable</h3>
    <ul>
      <li>
        <strong>Get row(s)</strong> — loads all images belonging to the album.
      </li>
      <li>
        <strong>Get row(s)1</strong> — retrieves the caption row.
      </li>
    </ul>
  </li>

  <li>
    <h3>Python image processor: assigning photo IDs</h3>
    <p>
      The <strong>Code in Python (Beta)</strong> node loops through buffered images and assigns:
    </p>
    <pre><code>id_fotos = 1, 2, 3, ...</code></pre>
  </li>

  <li>
    <h3>Python caption parser: extracting product data</h3>
    <p>This huge Python logic does:</p>

    <ul>
      <li>splits caption into product blocks by 5-digit keys (e.g. <code>00012</code>);</li>
      <li>extracts:
        <ul>
          <li>product number</li>
          <li>brand (smart detection of Latin words)</li>
          <li>colour & material (based on 100+ keywords)</li>
          <li>sizes (EU or letter, including ranges)</li>
          <li>new/old price</li>
          <li>raw description</li>
        </ul>
      </li>
      <li>expands size ranges:
        <li><code>38-40 → 38, 39, 40</code></li>
        <li><code>S–XL → S, M, L, XL</code></li>
      </li>
      <li>creates 1 item per size;</li>
      <li>assigns <code>id_item</code> to match photos.</li>
    </ul>
  </li>

  <li>
    <h3>Mapping products to photos</h3>
    <ul>
      <li><strong>Split Out</strong> — expands product array.</li>
      <li><strong>Split Out2</strong> — expands image array.</li>
      <li><strong>Merge1</strong> — matches:
        <pre>items.id_item == fotos.id_fotos</pre>
      </li>
    </ul>
  </li>

  <li>
    <h3>Downloading photo path from Telegram</h3>
    <p>
      <strong>Get a file</strong> retrieves <code>file_path</code>,  
      then a JavaScript node generates:
    </p>

    <pre><code>https://api.telegram.org/file/botTOKEN/file_path</code></pre>
  </li>

  <li>
    <h3>Creating Airtable product records</h3>
    <p>
      <strong>Create a record</strong> populates the Airtable table with:
    </p>
    <ul>
      <li>product_number</li>
      <li>Brand</li>
      <li>Size</li>
      <li>Price</li>
      <li>Colour</li>
      <li>Description</li>
      <li>Product Photo (direct URL)</li>
    </ul>
  </li>

  <li>
    <h3>Cleaning the DataTable buffer</h3>
    <p>
      <strong>Delete row(s)</strong> removes all temporary rows  
      associated with this processed album.
    </p>
  </li>

  <li>
    <h3>Final notification</h3>
    <p>
      The bot sends a Telegram message:
      <br /><strong>“Data successfully added to the database!”</strong>
    </p>
  </li>
</ol>

<hr />

<h2>🧩 Nodes and Integrations Used</h2>

<h3>Telegram</h3>
<ul>
  <li><code>Telegram Trigger</code></li>
  <li><code>Get a file</code> — fetches image paths</li>
  <li><code>Send a text message</code> — manager notification</li>
</ul>

<h3>Airtable</h3>
<ul>
  <li><code>Create a record</code> — writes fully structured product entries</li>
</ul>

<h3>DataTables</h3>
<ul>
  <li>Album buffering</li>
  <li>Intermediate storage of image metadata</li>
  <li>Controlled deletion after success</li>
</ul>

<h3>Custom Code</h3>
<ul>
  <li><strong>Python: image ID generator</strong></li>
  <li><strong>Python: advanced caption parser</strong></li>
  <li><strong>JavaScript: building direct Telegram image URL</strong></li>
</ul>

<hr />

<h2>⚙️ Installation & Setup</h2>

<ol>
  <li>Deploy n8n (self-hosted or cloud).</li>
  <li>Create a Telegram bot for the channel.</li>
  <li>Create a DataTable for buffering album photos.</li>
  <li>Create an Airtable base with required fields.</li>
  <li>Import this workflow JSON into n8n.</li>
  <li>Configure Telegram API credentials.</li>
  <li>Configure Airtable Personal Access Token.</li>
  <li>Activate the workflow.</li>
</ol>

<hr />

<h2>🛠 Customization Options</h2>

<ul>
  <li>Extend colour and fabric keyword dictionaries.</li>
  <li>Add automatic product category detection (LLM).</li>
  <li>Add duplicate detection in Airtable.</li>
  <li>Use separate bots for admin and notifications.</li>
</ul>
