# script
pipeline {
    agent any

    environment {
        CSV_FILE = 'path/to/your/file.csv'
        HTML_FILE = 'path/to/your/file.html'
        RECIPIENT_EMAIL = 'recipient@example.com'
    }

    stages {
        stage('Convert CSV to HTML') {
            steps {
                script {
                    // Read the CSV file
                    def csvContent = readFile(file: CSV_FILE)

                    // Convert CSV to HTML
                    def csvLines = csvContent.split('\n')
                    def headers = csvLines[0].split(',')
                    def rows = csvLines[1..-1]

                    def htmlContent = """
                        <html>
                        <head>
                            <style>
                                table {
                                    width: 100%;
                                    border-collapse: collapse;
                                }
                                th {
                                    background-color: blue;
                                    color: white;
                                    border: 1px solid black;
                                    padding: 8px;
                                    text-align: left;
                                }
                                td {
                                    border: 1px solid black;
                                    padding: 8px;
                                    text-align: left;
                                }
                            </style>
                        </head>
                        <body>
                            <table>
                                <thead>
                                    <tr>
                    """
                    headers.each { header ->
                        htmlContent += "<th>${header}</th>"
                    }
                    htmlContent += """
                                    </tr>
                                </thead>
                                <tbody>
                    """
                    rows.each { row ->
                        htmlContent += "<tr>"
                        row.split(',').each { cell ->
                            htmlContent += "<td>${cell}</td>"
                        }
                        htmlContent += "</tr>"
                    }
                    htmlContent += """
                                </tbody>
                            </table>
                        </body>
                        </html>
                    """

                    // Write the HTML content to a file
                    writeFile file: HTML_FILE, text: htmlContent
                }
            }
        }

        stage('Send Email') {
            steps {
                script {
                    emailext(
                        subject: 'CSV to HTML Report',
                        body: 'Please find the attached HTML report converted from CSV.',
                        to: RECIPIENT_EMAIL,
                        attachFiles: HTML_FILE,
                        mimeType: 'text/html'
                    )
                }
            }
        }
    }
}
