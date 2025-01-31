# Cold Fusion Technical Documentation

In an effort to migrate away from Cold Fusion we are documenting the current processes.

The Cold Fusion scripts are located on abish at `//abish/inetpub/wwwroot/library/new-books` `smb://abish/inetpub/wwwroot/library/new-books`

<h2 id="actions">Daily Actions/Triggers</h2>

Below are a list of actions that are triggered via HTTP requests.

![Actions list image](./images/actions-list.png)

| Task Name   | Interval         | HTTP Method | Endpoint                                                             | Description              |
| ----------- | ---------------- | ----------- | -------------------------------------------------------------------- | ------------------------ |
| ACQ Mail    | Every 30 minutes | GET         | [http://abish.byui.edu/library/new-books/index.cfm/process-acmail]() | Checks for new books     |
| ACQ Mail(1) | Every 2 hours    | GET         | [http://abish.byui.edu/library/new-books/index.cfm/send-acmail/]()   | Send email to requestors |
| INB         | Every 12 hours   | GET         | [http://abish.byui.edu/library/new-books/index.cfm/import]()         | Imports new books        |

## [ACQ Mail](#actions) - Checks for new books

Here is the flow for this script.

1. Initializes an output string with a message indicating the start of the process.
2. Retrieves new books from the acmail table using `variables.acmailService.check().`
    - ```sql
        SELECT * FROM acmail
      ```
3. Iterates over each book in the newBooks collection.
4. For each book, it appends the book's item number to the output.
5. Retrieves bibliographic information for the book using variables.acmailService.getBibInfo(book.bib_no).
6. Iterates over each bibliographic entry to extract ISBN numbers.
7. Uses a regular expression to find valid ISBN numbers in the bibliographic text.
8. If a valid ISBN is found, it searches for a matching purchase order line in the po_line table using variables.acmailService.findPOLine(isbn).
9. If a matching purchase order line is found:
    - Appends the title and internal note of the purchase order line to the output.
    - Extracts the email alias from the internal note.
    - Retrieves librarian information using variables.acmailService.getLibrarian(email_alias).
    - Checks if the book is already in the email queue using variables.acmailService.checkPendingAcmail(book.item_no).
    - If a librarian is found and the book is not already in the queue, it adds the book to the email queue using variables.acmailService.queueNewBook(email_alias, book.item_no, book.bib_no).
    - If no librarian is found, it logs a message indicating the book will be deleted without sending an email.
10. If no matching purchase order line is found, it logs a message indicating the PO item was not found.
11. Appends the ISBN to the output.
12. Deletes the book from the acmail table using variables.acmailService.deleteNewBook(book.item_no, book.bib_no).
13. Appends a completion message to the output.
14. Renders the output as HTML using variables.fw.renderData("html", output).

```cfml
function processAcmail(){
    var output = "Checking for new books... <br />";
    // First we need to check horizon for books that have been added to the acqmail table by the item table trigger that
    // have changed status from "t" to "i"
    var newBooks = variables.acmailService.check();
    for(book in newBooks){
        output = output & book.item_no & "<br />";
        // For each book we need to grab all of the ISBN numbers and try to do a look up in the po_line table for a match
        var bib_info = variables.acmailService.getBibInfo(book.bib_no);
        for(bib in bib_info){
            var isbn = bib.text;
            var match = REFind("(97(8|9))?\d{9}(\d|X)", isbn, 1, True);
            if(match.pos[1]+match.len[1] != 0){
                isbn = Mid(isbn, match.pos[1], match.len[1]);
                var po_line = variables.acmailService.findPOLine(isbn);
                if(po_line.recordCount > 0){
                    output = output & po_line.title & " " & po_line.internal_note & "<br />";

                    // Once we find the po_line we can pull in the internal note and try to match it to a librarian
                    var email_alias = '';
                    var noteArray = listToArray(trim(po_line.internal_note), ' ');
                    if(arraylen(noteArray) > 0){
                        email_alias = noteArray[1];
                    }else{
                        email_alias = trim(po_line.internal_note);
                    }
                    //var email_alias = listToArray(trim(po_line.internal_note))[1];

                    var librarian = variables.acmailService.getLibrarian(email_alias);
                    var existingBook = variables.acmailService.checkPendingAcmail(book.item_no);
                    output = output & librarian.name & " " & librarian.email & po_line.isbn &  "<br />";
                    // Before adding the book to the email queue, we need to make sure a librian was found and the book
                    // isn't already in the queue.
                    if(librarian.recordCount > 0 && existingBook.recordCount < 1){
                        // Go ahead and add the book to the email queue
                        variables.acmailService.queueNewBook(email_alias, book.item_no, book.bib_no);
                    }else{
                        output = output & "Requestor (Librarian) not found for " & email_alias & ". Book will be deleted with sending email.<br />";
                    }

                }else{
                    output = output & "PO item not found.<br />";
                }
            }
            output = output & isbn & "<br />";
        }
        // We have done everything we can so we can now delete the book from the acmail table in horizon
        variables.acmailService.deleteNewBook(book.item_no, book.bib_no);
    }
    output = output & "Done! <br />";
    // Logs can be found at C:\inetpub\wwwroot\library\new-books\acmail-logs
    //WriteLog (output, 'Information', 'yes', 'process');
    variables.fw.renderData( "html", output );
}
```

## [ACQ Mail(1)](#actions) - Send email to requestors

Here is the flow for this script.

1. Initializes an output string with a message indicating the start of the process.
2. Retrieves all pending books using `variables.acmailService.processPendingAcmail()`.
3. Initializes an empty structure booksByLibrarian to group books by librarian.
4. Iterates over each pending book:
    - Checks if the librarian's email alias exists in booksByLibrarian. If not, initializes an array for that alias.
    - Retrieves item details using `variables.acmailService.findItem(book.item_no)`.
    - Appends the item to the array corresponding to the librarian's email alias.
5. Initializes an empty string all_book_tables to store the summary table for the alert email.
6. Iterates over each librarian in booksByLibrarian:
    - Retrieves librarian details using `variables.acmailService.getLibrarian(trim(key))`.
    - Generates a book table view for the librarian using `variables.fw.view('acmail/book_table', {"items": "#booksByLibrarian[key]#"})`.
    - Generates an email body view for the librarian using `variables.fw.view('acmail/notification', {"book_table": book_table, "librarian": "#librarian#"})`.
    - Configures and sends an email to the librarian using mailerService.
    - Appends a message to the output indicating the email was sent.
    - Adds the current book table to the summary book table all_book_tables.
    - Iterates over each book in the librarian's array:
        - Appends the book's processed status to the output.
        - Deletes the book from the pending list using `variables.acmailService.deletePendingBook(book["item##"], book["bib##"])`.
7. Checks if all_book_tables contains content (length > 10):
    - Generates an alert email body view using `variables.fw.view('acmail/alert', {"all_book_tables": all_book_tables})`.
    - Configures and sends an alert email to acquisitions using mailerService.
8. Appends a completion message to the output.
9. Renders the output as HTML using `variables.fw.renderData("html", output)`.

```cfml
function sendAcmail(){

    var output = "Processing ACQ Mail... <br />";
    // First we just need to grab all of the pending books
    var pendingBooks = variables.acmailService.processPendingAcmail();
    var booksByLibrarian = structnew();
    // Next we need to loop through and group each book by librarian so they only get one email
    for(book in pendingBooks){
        if(!structKeyExists(booksByLibrarian, book.email_alias)){
            booksByLibrarian[book.email_alias] = arrayNew(1);
        }
        var item = variables.acmailService.findItem(book.item_no);
        arrayAppend(booksByLibrarian[book.email_alias], item);
    }
    // Instantiate the summary table for the alert email sent to acquisitions
    var all_book_tables = "";
    // Loop through each librarian, populate a table, add it to the email, and send the email
    for(key in booksByLibrarian){

        var librarian = variables.acmailService.getLibrarian(trim(key));
        var book_table = variables.fw.view('acmail/book_table', {"items": "#booksByLibrarian[key]#"});
        var email_body = variables.fw.view('acmail/notification', {"book_table": book_table, "librarian": "#librarian#"});

        mailerService = new mail();
        mailerService.setTo(librarian.email);
        //mailerService.setBcc("fackrellj@byui.edu");
        mailerService.setFrom("libacquisitions@byui.edu");
        mailerService.setFailTo("fackrellj@byui.edu");
        mailerService.setSubject('[ACQ Mail] The Enclosed Items Have Been Received');
        mailerService.setType("html");
        mailerService.send(body = email_body);

        output = output & "Email Sent to " & librarian.name & " for: <br />";
        // add the current book table to the summary book table
        all_book_tables = all_book_tables & '<p><span style="font-weight: bold;">Recipient:</span> ' & librarian.name & '</p>' & book_table;
        for(book in booksByLibrarian[key]){
            output = output & book.processed & "<br />";
            // Now that all of the books have been sent to the librarian, we can go ahead and delete them
            variables.acmailService.deletePendingBook(book["item##"], book["bib##"]);
        }
    }
    // Just check and make sure something was added to the summary and then send the summary email alert
    if(len(all_book_tables) > 10){
        var alert_email_body = variables.fw.view('acmail/alert', {"all_book_tables": all_book_tables});

        mailerService = new mail();
        mailerService.setTo("libacquisitions@byui.edu");
        //mailerService.setBcc("fackrellj@byui.edu");
        mailerService.setFrom("libacquisitions@byui.edu");
        mailerService.setFailTo("fackrellj@byui.edu");
        mailerService.setSubject('[ACQ Mail] The Enclosed Items Have Been Sent to the Requestors');
        mailerService.setType("html");
        mailerService.send(body = alert_email_body);
    }


    output = output & "Done! <br />";
    // Logs can be found at C:\inetpub\wwwroot\library\new-books\acmail-logs
    //WriteLog (output, 'Information', 'yes', 'process');
    variables.fw.renderData( "html", output );
}
```

## [INB](#actions) - Imports new books

Here is the flow for this script.

1. Calls variables.bookService.weeklyImport() to retrieve a list of books to be imported.
2. Initializes a count variable to track the number of new books imported and a message variable.
3. Iterates over each book in the list:
    - Checks if the book already exists in the database using variables.bookService.check(book.bib_no).
    - If the book does not exist (obook.recordCount < 1):
    - Sets the book's description to book.text if its length is greater than 5; otherwise, sets it to book.longtext.
    - Attempts to create the book in the database using variables.bookService.create(book). If successful, increments the count.
    - If the book already exists:
    - Creates a temporary structure temp to hold update data.
    - Extracts a valid ISBN from book.isbn using a regular expression.
    - If the extracted ISBN is not already in the existing book's add_isbn field and is valid, appends it to temp.add_isbn.
    - Updates the book's collection and author fields in temp.
    - Calls variables.bookService.update(temp) to update the book in the database.
4. Renders a success message with the count of new books imported using variables.fw.renderData("text", "Succesfully imported " & count & '!').

```cfml
function default( rc ){
    var books = variables.bookService.weeklyImport();
    var count = 0;
    var message = '';
    for(book in books){
        var obook = variables.bookService.check(book.bib_no);
        if(obook.recordCount < 1){
            if(len(book.text) > 5){
                book.description = book.text;
            }else{
                book.description = book.longtext;
            }
            if(variables.bookService.create(book) != False){
                count = count + 1;
            }
        }else{
            var temp = structNew();
            temp.bib_no = book.bib_no;
            var isbn = "-1";
            var match = REFind("(97(8|9))?\d{9}(\d|X)", book.isbn, 1, True);
            if(match.pos[1]+match.len[1] != 0){
                isbn = Mid(book.isbn, match.pos[1], match.len[1]);
            }
            if(Find(isbn, obook.add_isbn) == 0 and isbn != "-1"){
                temp.add_isbn = obook.add_isbn & '|' & isbn;
                temp.add_isbn = Right(temp.add_isbn,len(temp.add_isbn)-1);
                //variables.bookService.update(temp);
            }
            //if(len(obook.collection) < 1){
                temp.collection = book.collection;
                temp.author = book.author;
                variables.bookService.update(temp);
            //}
        }
    }
    variables.fw.renderData( "text", "Succesfully imported " & count & '!' );
}
```
