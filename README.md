# airtable-image-export
Export Airtable images from grid view


I use Airtable as a collobarative workspace for an ecommerce venture. The workflow involves one user half-way across the country uploading images for each new table entry, while I then extract those images to upload to different websites. Many Airtable users were looking for an image export solution, so I thought I'd share one method for exporting uploaded images out of an Airtable cell.

## Preliminary Setup
This solution involves using the Unix command line so it's suitable for Mac & Linux users. I'm on a Mac so all the examples here will use Mac-centric tools. You will need to download the `csvkit` package, which can be added from the command line after you've installed [the Homebrew package manager](https://brew.sh/):

`brew install csvkit`

We'll also need a unique custom ID for each row of our table. We can accomplish this in Airtable using a formula:

<p align="center">
  <img src="https://github.com/geopor/airtable-image-export/blob/main/Airtable-Create_Unique_ItemID.png">
</p>

This adds an incremented number to the value in the "SKU" cell to come up with a unique index for each row. In this table, each row is an item of clothing and we're interested in exporting its associated images.

We'll download a .csv export of our table and use that as the basis for extracting the images:

<p align="center">
  <img src="https://github.com/geopor/airtable-image-export/blob/main/Airtable-Export_to_CSV.png">
</p>

This ends up in our ~/Downloads folder and we're ready to start extracting our images.

## Extracting Airtable Images From The CSV Export

I'm using Keyboard Maestro to bundle all the commands logically together but it's not necessary. Even if you're not using Keyboard Maestro, the structure it provides will help visualize the overall automation.

Here's how the actual table looks (click for a larger image):

<p align="center">
  <img src="https://github.com/geopor/airtable-image-export/blob/main/Airtable-Grid_View.png">
</p>

The images we want reside in the "Attachments" column. Notice that the "ID" cell is selected in the image above. The automation will copy the value of the "ID" cell and use this value to lookup the appropriate row in the .csv export of the table.

<p align="center">
  <img src="https://github.com/geopor/airtable-image-export/blob/main/Airtable-Keyboard_Maestro_Script.png">
</p>

Going through each numbered step in the image:

### 1. Copy The ID
We use Keyboard Maestro to copy the selected ID cell. Again, you don't technically need Keyboard Maestro to do this. Instead of assigning the value of the clipboard to a Keyboard Maestro variable (beginning with `$KMVAR`), a shell variable could be assigned using `pbpaste` on the command line if you don't wish to use the Keyboard Maestro app.

### 2. Set the Item_ID Variable
We set the `Item_ID` variable to the value we retrieved earlier from the clipboard. This will be used to search the .csv export for the relevant row as well as create a directory with the `Item_ID` in its name to hold our images.

### 3. Execute Shell Script - Set Variable "MostRecentCSV"
`ls -t ~/Downloads | grep ".*\.csv$" | head -1 |  tr " " "_"`

This shell command sorts all items in our ~/Downloads folder with the most recent at the top. That means when we `grep` for a .csv file, it will match our most recent export. The last pipe command `tr` replaces spaces in the filename with underscores.

### 4. Execute Shell Script - Make Directory & Find Images in CSV
```
cd ~/Downloads/_item_pics
mkdir $KMVAR_Item_ID
cd $KMVAR_Item_ID
csvgrep -c ID -m $KMVAR_Item_ID ~/Downloads/$KMVAR_MostRecentCSV  | csvcut -c 6 | awk -F "[()]" '{ for (i=2; i<NF; i+=2) print $i }' | xargs -n 1 curl -O 
```
The last shell script we execute will create a folder in our exported images base folder to hold our images (here we're using `~/Downloads/_item_pics`) . We'll `cd` into the directory so that the output of the next command will be sent to this folder.

`csvgrep` searches for the item's ID (`$KMVAR_ItemID`) in the most recently downloaded csv export (`$KMVAR_MostRecentCSV`). The output is the found row, of which we only want the sixth column containing our images (`csvcut -c 6`). The `awk` and `print` commands strip away unecessary characters and return only the raw image URLs. Finally, `xargs` is called to loop through each image and `curl` it into our local folder.

### 5. Open The Resulting Folder
We have Keyboard Maestro open the folder containing our exported images. You can customize this script and add to it if you need to perform other actions with this script. For example, the title of the item can be saved to a variable and automatically entered into a webpage that the script loads. 

You can also export all images from your table in one go by simply going through each row one by one and running `xargs` and `curl` on the raw image URLs. For example, given a latest `table-latest.csv` Airtable export where the attachments are in the 6th column, this command will download all images for the table in the current directory:

`cat table-latest.csv | csvcut -c 6 | awk -F "[()]" '{ for (i=2; i<NF; i+=2) print $i }' | xargs -n 1 curl -O` 
