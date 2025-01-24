# Cold Fusion Technical Documentation

In an effort to migrate away from Cold Fusion we are documenting the current processes.

The Cold Fusion scripts are located on abish at `smb://abish/inetpub`

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
