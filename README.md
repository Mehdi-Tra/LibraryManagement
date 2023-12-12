using System;
using System.Collections.Generic;
using System.Data;
using System.Data.SQLite;
using System.IO;
using System.Text;

public class Book
{
    public string ISBN { get; set; }
    public string Title { get; set; }
    public Author Author { get; set; }
    public int PublicationYear { get; set; }
    public string Genre { get; set; }
}

public class Author
{
    public int AuthorId { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public DateTime DateOfBirth { get; set; }
}

public class Borrowing
{
    public int BorrowingId { get; set; }
    public Book BorrowedBook { get; set; }
    public DateTime BorrowDate { get; set; }
    public DateTime ExpectedReturnDate { get; set; }
    public string Status { get; set; }
}

public class LibraryManagement
{
    private const string DatabaseFileName = "LibraryManagement.db";
    private const string ConnectionString = "Data Source=" + DatabaseFileName + ";Version=3;";

    public List<Book> Books { get; private set; }
    public List<Author> Authors { get; private set; }
    public List<Borrowing> Borrowings { get; private set; }

    public LibraryManagement()
    {
        InitializeDatabase();
        LoadData();
    }

    private void InitializeDatabase()
    {
        if (!File.Exists(DatabaseFileName))
        {
            SQLiteConnection.CreateFile(DatabaseFileName);
            using (var connection = new SQLiteConnection(ConnectionString))
            {
                connection.Open();

                // Créer les tables
                ExecuteNonQuery(connection, "CREATE TABLE Books (ISBN TEXT PRIMARY KEY, Title TEXT NOT NULL, AuthorId INTEGER, PublicationYear INTEGER, Genre TEXT);");
                ExecuteNonQuery(connection, "CREATE TABLE Authors (AuthorId INTEGER PRIMARY KEY, FirstName TEXT NOT NULL, LastName TEXT NOT NULL, DateOfBirth TEXT);");
                ExecuteNonQuery(connection, "CREATE TABLE Borrowings (BorrowingId INTEGER PRIMARY KEY, ISBN TEXT, BorrowDate TEXT, ExpectedReturnDate TEXT, Status TEXT);");

                // Ajouter des contraintes de clé étrangère
                ExecuteNonQuery(connection, "ALTER TABLE Books ADD FOREIGN KEY (AuthorId) REFERENCES Authors(AuthorId);");
                ExecuteNonQuery(connection, "ALTER TABLE Borrowings ADD FOREIGN KEY (ISBN) REFERENCES Books(ISBN);");
            }
        }
    }

    private void LoadData()
    {
        Books = LoadBooks();
        Authors = LoadAuthors();
        Borrowings = LoadBorrowings();
    }

    private List<Book> LoadBooks()
    {
        var books = new List<Book>();
        using (var connection = new SQLiteConnection(ConnectionString))
        {
            connection.Open();
            using (var command = new SQLiteCommand("SELECT * FROM Books", connection))
            using (var reader = command.ExecuteReader())
            {
                while (reader.Read())
                {
                    var book = new Book
                    {
                        ISBN = reader["ISBN"].ToString(),
                        Title = reader["Title"].ToString(),
                        Author = GetAuthorById(Convert.ToInt32(reader["AuthorId"])),
                        PublicationYear = Convert.ToInt32(reader["PublicationYear"]),
                        Genre = reader["Genre"].ToString()
                    };
                    books.Add(book);
                }
            }
        }
        return books;
    }

    private List<Author> LoadAuthors()
    {
        var authors = new List<Author>();
        using (var connection = new SQLiteConnection(ConnectionString))
        {
            connection.Open();
            using (var command = new SQLiteCommand("SELECT * FROM Authors", connection))
            using (var reader = command.ExecuteReader())
            {
                while (reader.Read())
                {
                    var author = new Author
                    {
                        AuthorId = Convert.ToInt32(reader["AuthorId"]),
                        FirstName = reader["FirstName"].ToString(),
                        LastName = reader["LastName"].ToString(),
                        DateOfBirth = DateTime.Parse(reader["DateOfBirth"].ToString())
                    };
                    authors.Add(author);
                }
            }
        }
        return authors;
    }

    private List<Borrowing> LoadBorrowings()
    {
        var borrowings = new List<Borrowing>();
        using (var connection = new SQLiteConnection(ConnectionString))
        {
            connection.Open();
            using (var command = new SQLiteCommand("SELECT * FROM Borrowings", connection))
            using (var reader = command.ExecuteReader())
            {
                while (reader.Read())
                {
                    var borrowing = new Borrowing
                    {
                        BorrowingId = Convert.ToInt32(reader["BorrowingId"]),
                        BorrowedBook = GetBookByISBN(reader["ISBN"].ToString()),
                        BorrowDate = DateTime.Parse(reader["BorrowDate"].ToString()),
                        ExpectedReturnDate = DateTime.Parse(reader["ExpectedReturnDate"].ToString()),
                        Status = reader["Status"].ToString()
                    };
                    borrowings.Add(borrowing);
                }
            }
        }
        return borrowings;
    }

    private Author GetAuthorById(int authorId)
    {
        return Authors.Find(author => author.AuthorId == authorId);
    }

    private Book GetBookByISBN(string isbn)
    {
        return Books.Find(book => book.ISBN == isbn);
    }

    public void AddBook(Book book)
    {
        using (var connection = new SQLiteConnection(ConnectionString))
        {
            connection.Open();
            using (var command = new SQLiteCommand(connection))
            {
                command.CommandText = "INSERT INTO Books (ISBN, Title, AuthorId, PublicationYear, Genre) VALUES (@isbn, @title, @authorId, @publicationYear, @genre);";
                command.Parameters.AddWithValue("@isbn", book.ISBN);
                command.Parameters.AddWithValue("@title", book.Title);
                command.Parameters.AddWithValue("@authorId", book.Author.AuthorId);
                command.Parameters.AddWithValue("@publicationYear", book.PublicationYear);
                command.Parameters.AddWithValue("@genre", book.Genre);
                command.ExecuteNonQuery();
            }
        }
        Books.Add(book);
    }

    public void AddAuthor(Author author)
    {
        using (var connection = new SQLiteConnection(ConnectionString))
        {
            connection.Open();
            using (var command = new SQLiteCommand(connection))
            {
                command.CommandText = "INSERT INTO Authors (AuthorId, FirstName, LastName, DateOfBirth) VALUES (@authorId, @firstName, @lastName, @dateOfBirth);";
                command.Parameters.AddWithValue("@authorId", author.AuthorId);
                command.Parameters.AddWithValue("@firstName", author.FirstName);
                command.Parameters.AddWithValue("@lastName", author.LastName);
                command.Parameters.AddWithValue("@dateOfBirth", author.DateOfBirth.ToString("yyyy-MM-dd"));
                command.ExecuteNonQuery();
            }
        }
        Authors.Add(author);
    }

    public void AddBorrowing(Borrowing borrowing)
    {
        using (var connection = new SQLiteConnection(ConnectionString))
        {
            connection.Open();
            using (var command = new SQLiteCommand(connection))
            {
                command.CommandText = "INSERT INTO Borrowings (ISBN, BorrowDate, ExpectedReturnDate, Status) VALUES (@isbn, @borrowDate, @expectedReturnDate, @status);";
                command.Parameters.AddWithValue("@isbn", borrowing.BorrowedBook.ISBN);
                command.Parameters.AddWithValue("@borrowDate", borrowing.BorrowDate.ToString("yyyy-MM-dd"));
                command.Parameters.AddWithValue("@expectedReturnDate", borrowing.ExpectedReturnDate.ToString("yyyy-MM-dd"));
                command.Parameters.AddWithValue("@status", borrowing.Status);
                command.ExecuteNonQuery();
            }
        }
        Borrowings.Add(borrowing);
    }

    public void GenerateBookFile(Book book)
    {
        var fileName = $"{book.Title}_{book.ISBN}.txt";
        var fileContent = new StringBuilder();
        fileContent.AppendLine($"Title: {book.Title}");
        fileContent.AppendLine($"ISBN: {book.ISBN}");
        fileContent.AppendLine($"Author: {book.Author.FirstName} {book.Author.LastName}");
        fileContent.AppendLine($"Publication Year: {book.PublicationYear}");
        fileContent.AppendLine($"Genre: {book.Genre}");
        fileContent.AppendLine("Borrowing History:");
        foreach (var borrowing in Borrowings)
        {
            if (borrowing.BorrowedBook.ISBN == book.ISBN)
            {
                fileContent.AppendLine($"- Borrowed on {borrowing.BorrowDate.ToString("yyyy-MM-dd")}, Expected Return: {borrowing.ExpectedReturnDate.ToString("yyyy-MM-dd")}, Status: {borrowing.Status}");
            }
        }

        File.WriteAllText(fileName, fileContent.ToString());
    }

    private void ExecuteNonQuery(SQLiteConnection connection, string sql)
    {
        using (var command = new SQLiteCommand(sql, connection))
        {
            command.ExecuteNonQuery();
        }
    }
}

class Program
{
    static void Main()
    {
        var libraryManagement = new LibraryManagement();

        // Exemple d'utilisation
        var author = new Author { AuthorId = 1, FirstName = "John", LastName = "Doe", DateOfBirth = new DateTime(1980, 1, 1) };
        var book = new Book { ISBN = "1234567890", Title = "Sample Book", Author = author, PublicationYear = 2022, Genre = "Fiction" };
        var borrowing = new Borrowing { BorrowedBook = book, BorrowDate = DateTime.Now, ExpectedReturnDate = DateTime.Now.AddDays(14), Status = "En cours" };

        libraryManagement.AddAuthor(author);
        libraryManagement.AddBook(book);
        libraryManagement.AddBorrowing(borrowing);

        // Affichage des livres
        Console.WriteLine("Books in the library:");
        foreach (var b in libraryManagement.Books)
        {
            Console.WriteLine($"{b.Title} - ISBN: {b.ISBN}");
        }

        // Affichage des auteurs
        Console.WriteLine("\nAuthors in the library:");
        foreach (var a in libraryManagement.Authors)
        {
            Console.WriteLine($"{a.FirstName} {a.LastName} - ID: {a.AuthorId}");
        }

        // Affichage des emprunts
        Console.WriteLine("\nBorrowings in the library:");
        foreach (var b in libraryManagement.Borrowings)
        {
            Console.WriteLine($"{b.BorrowedBook.Title} - Borrowed on {b.BorrowDate.ToString("yyyy-MM-dd")}, Expected Return: {b.ExpectedReturnDate.ToString("yyyy-MM-dd")}, Status: {b.Status}");
        }

        // Génération d'un fichier pour un livre
        libraryManagement.GenerateBookFile(book);
        Console.WriteLine("\nBook file generated.");
    }
}
