Wishlist Application Report

Introduction

This report details the implementation of a web application that manages children’s wishlists. The application interacts with a relational database using PHP and PDO with prepared statements. It allows users to add new wishlists, view and evaluate existing wishlists, deliver wishlists, and calculate the average kindness level per year. Each implemented feature is discussed with corresponding code examples and explanations.

Link to Working Web Page

[HAVE DONE] (Note: The working web page is uploaded on the school’s web server as required.)

Form with Free Text Fields to Add Data to Tables

Implementation

The feature is implemented in the add_data.php file. It presents a form where users can input the Social Security Number (SSN) of a child and specify the number of toys to add to the wishlist.

Figure 1: Form with Free Text Fields in add_data.php

<form action="add_data.php" method="post">
    <h3><label>SSN:</label>
        <input type="text" name="ssn" value="201101-1234" required><br>
    </label></h3>

    <label>Number of toys:</label>
        <input type="number" name="number" value="1" required><br><br>
    </label>
    <input type="submit" value="Continue">
</form>

Explanation

	•	SSN Field: A text input field where the user enters the child’s SSN.
	•	Number of Toys Field: A number input field where the user specifies how many toys they wish to add.
	•	Both fields are marked as required to ensure the form is submitted with complete information.

Dropdown Generated from Data in the Database

Implementation

After the user submits the initial form, add_data.php generates dropdown menus populated with toy names fetched from the database.

Figure 2: Generating Dropdown Menus in add_data.php

$pdo = new PDO('mysql:host=localhost;dbname=a23petny', $_SESSION['username'], $_SESSION['password']);

$stmt = $pdo->prepare("SELECT namn FROM leksak");
$stmt->execute();
$result = $stmt->fetchAll(PDO::FETCH_ASSOC);

for ($i = 1; $i < $_POST["number"]+1; $i++) {
    echo "<label>Toy {$i}:
        <select name='decision{$i}' required>
            <option value=''>--Select--</option>";
    foreach ($result as $row) {
        echo "<option value='{$row["namn"]}'>{$row["namn"]}</option>\n";
    }
    echo "</select>
    </label><br>";
}

Explanation

	•	Database Connection: Establishes a connection to the database using PDO.
	•	Fetching Data: Retrieves all toy names from the leksak table.
	•	Generating Dropdowns: For each toy number specified, a dropdown menu is created with options populated from the database.
	•	Looping: Uses a for loop to generate the required number of dropdowns based on user input.

Search in the Database with Free Text Search

Implementation

Implemented in avg_kindness.php, this feature allows users to input a year and retrieve the average kindness level of children born that year.

Figure 3: Search Form in avg_kindness.php

<form action="avg_kindness.php" method="get">
    <label>Year: 
        <input type='text' name="year">
    </label>
    <input type='submit' value='Get Data'>
</form>

Figure 4: Executing Stored Procedure in avg_kindness.php

if (isset($_GET["year"])) {
    $pdo = new PDO('mysql:host=localhost;dbname=a23petny', $_SESSION['username'], $_SESSION['password']);
    $stmt = $pdo->prepare("CALL GetAvgSnällhetForYear(:year);");
    $stmt->execute([':year' => $_GET['year']]);
    
    $result = $stmt->fetch(PDO::FETCH_ASSOC);
    if ($result) {
        echo 'For the year ' . $_GET['year'] . ' the average kindness was ' . $result['AvgSnällhetNivå'];
    } else {
        echo 'No information for the year ' . $_GET['year'];
    }
}

Explanation

	•	Form Submission: The user inputs a year, and upon submission, the form sends a GET request.
	•	Stored Procedure Call: The PHP script checks if a year is set and calls the GetAvgSnällhetForYear stored procedure with the input year.
	•	Displaying Results: Fetches the result and displays the average kindness level or a message if no data is available.

Change to the Contents of the Database Using UPDATE

Implementation

Users can approve or reject wishlists in view_wishlists.php, which updates the database accordingly.

Figure 5: Updating Database in view_wishlists.php

if (isset($_POST['decision'])) {
    if ($_POST['decision'] == 'approved') {
        $stmt = $pdo->prepare("UPDATE önskelista SET medgiven = 2 WHERE ssn = :ssn AND årtal = :year");
    } elseif ($_POST['decision'] == 'rejected') {
        $stmt = $pdo->prepare("UPDATE önskelista SET medgiven = 0 WHERE ssn = :ssn AND årtal = :year");
    }
    $stmt->execute([
        ':ssn' => $_POST['ssn'],
        ':year' => date('Y')
    ]);
}

Explanation

	•	Decision Handling: Checks the user’s decision from the form submission.
	•	Updating önskelista Table: Uses an UPDATE statement to set the medgiven status based on the decision.
	•	2 represents an approved wishlist.
	•	0 represents a rejected wishlist.
	•	Execution: Prepares and executes the SQL statement with bound parameters to prevent SQL injection.

Execute a Procedure from the Database

Implementation

The avg_kindness.php page calls a stored procedure to retrieve data.

Figure 6: Calling Stored Procedure in avg_kindness.php

$stmt = $pdo->prepare("CALL GetAvgSnällhetForYear(:year);");
$stmt->execute([':year' => $_GET['year']]);

Explanation

	•	Stored Procedure: GetAvgSnällhetForYear calculates the average kindness level for a given year.
	•	Parameter Binding: The year input by the user is bound to the procedure call.
	•	Result Handling: The result is fetched and displayed to the user.

Display the Contents from Two Different Database Tables

Implementation

In delivered.php, the contents of önskelista and leveradÖnskelista tables are displayed.

Figure 7: Displaying Undelivered Wishlists in delivered.php

foreach ($pdo->query("SELECT * FROM önskelista") as $row) {
    // Display undelivered wishlist details
}

Figure 8: Displaying Delivered Wishlists in delivered.php

foreach ($pdo->query("SELECT * FROM leveradÖnskelista") as $row) {
    // Display delivered wishlist details
}

Explanation

	•	Undelivered Wishlists: Retrieved from the önskelista table.
	•	Delivered Wishlists: Retrieved from the leveradÖnskelista table.
	•	Separation: Wishlists are categorized and displayed under appropriate headings.

Generate Links Based on Database Content

Implementation

Links are generated in delivered.php, allowing users to mark wishlists as delivered.

Figure 9: Generating Links in delivered.php

echo "<li><a href='delivered.php?ssn={$row['ssn']}&årtal={$row['årtal']}'>
    wishlist by {$namn}, ssn: {$row['ssn']}, Year: {$row['årtal']}, Responsible: {$row['ansvarigHandläggare']}
</a></li>";

Figure 10: Executing Procedure on Link Click in delivered.php

if (isset($_GET["ssn"])) {
    $stmt = $pdo->prepare("CALL levererad(:ssn, :year)");
    $stmt->execute([
        ':ssn' => $_GET['ssn'],
        ':year' => $_GET["årtal"]
    ]);
}

Explanation

	•	Link Generation: Each wishlist is a clickable link that includes the child’s SSN and year.
	•	Procedure Execution: Upon clicking the link, the levererad stored procedure is called to move the wishlist to the delivered table.
	•	Database Update: The procedure handles data movement and deletion as required.

VG: Implement a Login that Checks Credentials Against the Database

Implementation

A login system is implemented using login.php and login_check.php.

Figure 11: Login Form in login.php

<form action="login_check.php" method="post">
    <label>Username:</label>
        <input type="text" name="username" value="root" required><br><br>
    </label>

    <label>Password:
        <input type="password" name="password" value="new_password" required><br><br>
    </label>
    <input type="submit" value="Login">
</form>

Figure 12: Login Verification in login_check.php

try {
    $pdo = new PDO('mysql:host=localhost;dbname=a23petny', $_POST['username'], $_POST['password']);
    if (isset($_POST['username'])) {
        $_SESSION["username"] = $_POST['username'];
    }
    if (isset($_POST['password'])) {
        $_SESSION["password"] = $_POST['password'];
    }
    header('Location: main_menu.php');
} catch (PDOException $e) {
    echo "<br><br>" . $e->getMessage();
}

Explanation

	•	Login Form: Collects username and password from the user.
	•	Session Handling: Stores credentials in the session upon successful login.
	•	Database Connection: Attempts to connect to the database using provided credentials.
	•	Error Handling: Catches exceptions and displays an error message if login fails.

VG: Implement a Procedure That Receives a Parameter via the Interface

Implementation

The GetAvgSnällhetForYear stored procedure receives a year parameter from user input.

Figure 13: Stored Procedure Call in avg_kindness.php

$stmt = $pdo->prepare("CALL GetAvgSnällhetForYear(:year);");
$stmt->execute([':year' => $_GET['year']]);

Explanation

	•	User Input: The year is entered by the user in a text field.
	•	Parameter Passing: The input year is passed as a parameter to the stored procedure.
	•	Procedure Execution: Retrieves the average kindness level for the specified year.

VG: Create a Form That Uses Hidden Fields

Implementation

Hidden fields are used in add_data.php to pass data between form submissions.

Figure 14: Hidden Fields in add_data.php

<input type="hidden" name="ssn" value="<?php echo isset($_POST["ssn"]) ? $_POST["ssn"] : ''; ?>">
<input type="hidden" name="number" value="<?php echo isset($_POST["number"]) ? $_POST["number"] : ''; ?>">

Explanation

	•	Data Preservation: Keeps the SSN and number of toys across multiple form submissions.
	•	Multi-step Form: Facilitates a form that progresses in steps, retaining necessary data.
	•	Conditional Value: Checks if the POST variables are set before assigning them.

Conclusion

The application fulfills all the required functionalities, including:

	•	Adding data through forms with free text fields.
	•	Generating dropdowns from database data.
	•	Searching the database using user input.
	•	Updating and deleting database content.
	•	Executing stored procedures.
	•	Displaying data from multiple tables.
	•	Generating dynamic links that trigger database changes.

Additional VG (Very Good) requirements met include:

	•	Implementing a secure login system using PHP sessions.
	•	Executing stored procedures with parameters supplied via the interface.
	•	Using hidden fields in forms to handle multi-step data submission.

References

	•	PHP PDO Documentation: PHP Manual - PDO
	•	MySQL Stored Procedures: MySQL Documentation - Stored Procedures
	•	HTML Forms: MDN Web Docs - HTML Forms

Note: All code examples are presented in monospace font with proper indentation for readability. Each code snippet is labeled as a numbered figure for reference.
