## Κύρια συμπεράσματα μετά την ανάγνωση της δομής των BMP αρχείων

1. Τα στοιχεία που πρέπει να αλλάξω στα *Headers* για να αναπαρασταθεί η περιστραμμένη εικόνα είναι:

    | **Θέση bytes** | **Πληροφορία που περιγράφουν** |
    |:--------------:|:------------------------------:|
    | 2~4 bytes      | Μέγεθος αρχείου (σε bytes)     |
    | 18~22 bytes    | Πλάτος εικόνας (σε pixels)     |
    | 22~26 bytes    | Ύψος εικόνας (σε pixels)       |
    | 34~38 bytes    | Μέγεθος εικόνας (σε bytes)     |
    | 38~42 bytes    | Horizontal resolution          |
    | 42~46 bytes    | Vertical resolution            |

2. Τα *Other Data*, με τα οποία σύμφωνα με την εκφώνηση της άσκησης δεν ασχολούμαι, απλά θα τα αντιγράψω γνωρίζοντας ότι βρίσκονται μετά τα 54 bytes των *Headers* και πριν το *offset* του *2D pixel array*.

3. Το *Padding* είναι ένας αριθμός από bytes που:
    - συμπληρώνει την κάθε γραμμή του *2D pixel array* ώστε το συνολικό μέγεθος (σε bytes) της κάθε γραμμής να είναι πολλαπλάσιο του 4
    - εισάγεται στο τέλος της κάθε γραμμής αλλά δεν συμβάλλει στον υπολογισμό του πλάτους της εικόνας (παρόλο που εξαιτίας του αλλάζει το μέγεθος της εικόνας)
    - μπορεί να υπολογιστεί αν από το 4 αφαιρέσω το υπόλοιπο της ακέραιας διαίρεσης με το 4,  του *Width* επί το 3 (μέγεθος κάθε pixel σε bytes). Σε περίπτωση που το *Width* επί το 3 είναι πολλαπλάσιο του 4 τότε το *Padding* είναι ίσο με 0:

    ```c
    padding = 4 - ((width * 3) % 4);
        if(padding == 4) padding = 0;
    ```

----

## Βασική ιδέα για την δημιουργία περιστραμμένης εικόνας

#### Αντιγραφή εισόδου σε πίνακα `og_image[]`

1. Αφού η βασική είσοδος του προγράμματος είναι το ***stdin*** και δεν μπορώ να διαχειριστώ τα δεδομένα εισόδου ως αρχεία (δηλαδή δεν μπορώ να χρησιμοποιήσω την `fopen()`) ένας σίγουρος τρόπος για την επεξεργασία των δεδομένων εισόδου είναι η αντιγραφή των δεδομένων αυτών σε έναν πίνακα με την `fread()` από το ***stdin***.

2. Από την στιγμή που το μέγεθος του αρχείου αναγράφεται σε little endian στα bytes 2~4 πρέπει πρώτα να διαβαστούν τα πρώτα 2 bytes και έπειτα τα επόμενα 4 να αντιγραφούν σε μεταβλητή τύπου int. Έτσι βρίσκω το μέγεθος του πίνακα τύπου char που θα αρχικοποιήσω και την μνήμη που θα δεσμεύσω δυναμικά:
    <details>
    <summary>
    Μεταβλητές
    </summary>

    | **Τύπος** | **Όνομα**         | **Χρήση**                                          |
    |:---------:|:-----------------:|:--------------------------------------------------:|
    | *char* *  | file_size_bytes   | δείχνει σε κάθε ένα byte του file_size             |
    | *char*    | B                 | αποθήκευση πρώτου byte και έλεγχος αν είναι το Β   |
    | *char*    | M                 | αποθήκευση δεύτερου byte και έλεγχος αν είναι το M |
    | *int*     | file_size         | αποθήκευση μεγέθους αρχείου                        |
    | *int*     | bytes_read        | έλεγχος επιτυχίας ανάγνωσης των bytes με `fread()` |

    </details>

    <details>
    <summary> 
    Κώδικας για δέσμευση og_image
    </summary>

    ```c
    bytes_read = fread(&B, 1, 1, stdin);
    if(bytes_read != 1){
        fprintf(stderr, "Error: couldn't read the BM initial\n");
        exit(1);
    }
    if(B != 'B'){
        fprintf(stderr,"Error: not a BMP file\n");
        exit(1);
    }
    bytes_read = fread(&M, 1, 1, stdin);
    if(bytes_read != 1){
        fprintf(stderr,"Error: couldn't read the BM initial\n");
        exit(1);
    }
    if(M != 'M'){
        fprintf(stderr,"Error: not a BMP file\n");
        exit(1);
    } 

    bytes_read = fread(&file_size, 4, 1, stdin);
    if(bytes_read != 1){
        fprintf(stderr,"Error: couldn't read the file size from header\n");
        exit(1);
    }

    char *og_image = malloc((file_size) * sizeof(char));
    if(og_image == NULL){
        fprintf(stderr, "Error while allocating memory");
        exit(1);  
    }
    ```

    </details>



3. Η χρήση της `fseek()` οδηγεί σε *undefined behavior* καθώς δεν έχω συνδέσει το αρχείο στο πρόγραμμα μέσω `fopen()`. Έτσι δεν μπορώ να ξαναδιαβάσω τα 6 πρώτα bytes του αρχείου, άρα απλά θα τα αντιγράψω στο `og_image[]` με χρήση των προηγούμενων μεταβλητών.
    - Οι μεταβλητές **B** και **M** μπορούν να αντιγραφούν με απλό τρόπο καθώς είναι τύπου *char* και έχουν μέγεθος 1 byte: 
        <details>
        <summary></summary>

        ```c
        og_image[0] = 'B';
        og_image[1] = 'M';
        ```

        </details>
    - Η μεταβλητή **file_size** όμως είναι τύπου *int* έχει μέγεθος 4 bytes και πρέπει να αποθηκευτεί στον πίνακα το κάθε byte της σε μορφή little endian: 
        <details>
        <summary></summary>

        ```c
        file_size_bytes = (char*) &file_size;
        og_image[2] = file_size_bytes[0];
        og_image[3] = file_size_bytes[1];
        og_image[4] = file_size_bytes[2];
        og_image[5] = file_size_bytes[3];
        ```

        </details>

4. Μετά την δέσμευση του χώρου για τον πίνακα αποθηκεύω όλα τα υπόλοιπα δεδομένα εισόδου από το ***stdin*** (file_size - 6) στο `og_image[]` ξεκινώντας από το 7ο στοιχείο του πίνακα. Αν η `fread()` αποτύχει την ανάγνωση των στοιχείων τυπώνεται ανάλογο μήνυμα στην οθόνη με `fprintf()` στο ***stderr***. Αν το αρχείο είναι μικρότερο των 54 bytes δεν είναι δεκτή είσοδος του προγράμματος: 
    <details>
    <summary></summary>

    ```c
    bytes_read = fread(og_image+6, 1, file_size-6, stdin);
    if(bytes_read != file_size-6){
        fprintf(stderr,"Error: couldn't read the BMP file\n");
        exit(1);
    }
    if(bytes_read + 6 < 54){
        fprintf(stderr,"Error: file size has to be at least 54 bytes");
        exit(1);
    }
    ```

    </details>

#### Αρχικοποίηση μεταβλητών που θα χρησιμιποιηθούν για την δημιουργία του `rotated_image[]`

1. <details>
    <summary>Μεταβλητές</summary>

    ||***Αρχηκή Εικόνα***||
    |:---------:|:---------------------:|:-------------------------:|
    | **Τύπος** | **Όνομα**             | **Χρήση**                 |
    | *int*     | height                | ύψος εικόνας              |
    | *int*     | width                 | πλάτος εικόνας            |
    | *int*     | padding               | padding εικόνας           |
    | *int*     | pixel2d_array_start   | offset του 2D pixel array |
    | *int*     | horizontal_resolution | οριζόντια ανάλυση         |
    | *int*     | vertical_resolution   | κάθετη ανάλυση            |
    | *int*     | dib_header_size       | μέγεθος DIB Header        |
    | *int*     | image_size            | μέγεθος εικόνας           |

    <br></br>

    ||***Περιστραμμένη εικόνα***||
    |:---------:|:-------------------------:|:-----------------------------:|
    | **Τύπος** | **Όνομα**                 | **Χρήση**                     |
    | *int*     | new_height                | νέο ύψος εικόνας              |
    | *int*     | new_width                 | νέο πλάτος εικόνας            |
    | *int*     | new_padding               | νέο padding εικόνας           |
    | *int*     | new_file_size             | νέο offset του 2D pixel array |
    | *int*     | new_horizontal_resolution | νέα οριζόντια ανάλυση         |
    | *int*     | new_vertical_resolution   | νέα κάθετη ανάλυση            |
    | *int*     | new_image_size            | νέο μέγεθος εικόνας           |

    </details>

2. Φτιάχνω την συνάρτηση `int copy_array(char *array, int byte_number)` για την αντιγραφή `byte_number` στοιχείων  του πίνακα τύπου *char* `*array` σε μεταβλητή τούπου *int* `integer`. Με τον δείκτη σε κάθε byte του `integer` και μια `for` επιτυγχάνω αυτήν την αντιγραφή και μετά το τέλος της επιστρέφω την τιμή του `integer`: 
    <details>
    <summary>Συνάρτηση</summary>

    ```c
    int copy_array(char *array, int byte_number){
    int i, integer;
    char *ch_array;
    ch_array = (char*) &integer;

    for(i=0; i<4; i++){
        ch_array[i] = array[byte_number];
        byte_number ++;
    }

    return integer;
    }
    ```

    </details>

3. Οι αρχικοποιείσεις γίνονται ως εξής:
    - Όλες οι μεταβλητές της αρχικής εικόνας εκτός του *padding* αρχικοποιούνται με την βοήθεια της `copy_array()`:
        <details>
        <summary></summary>

        ```c
        height = copy_array(og_image, 22);

        width = copy_array(og_image, 18);

        pixel2d_array_start = copy_array(og_image, 10);

        dib_header_size = copy_array(og_image, 14);

        image_size = copy_array(og_image, 34);

        horizontal_resolution = copy_array(og_image, 38);

        vertical_resolution = copy_array(og_image, 42);
        ```

        </details>

    - Κάποιες μεταβλητές του νέου πίνακα `rotated_image[]` της περιστραμμένης εικόνας παίρνουν τιμή με την αλλαγή των θέσεων μεταβλητών της αρχικής εικόνας σχετιζόμενες μεταξύ τους (για παράδειγμα width και length): 
        <details>
        <summary></summary>

        ```c
        new_height = width;
        new_width = height;

        new_horizontal_resolution = vertical_resolution;
        new_vertical_resolution = horizontal_resolution;
        ```

        </details>
    
    - Οι μεταβλητές *new_image_size* και *new_file_size* υπολογίζονται από τύπο:
        <details>
        <summary></summary>

        ```c
        new_image_size = (new_padding * new_height) + ((new_width * 3) * new_height);

        new_file_size = (new_padding * new_height) + ((new_width * 3) * new_height) + pixel2d_array_start;
        ```

        </details>
    
    - Γίνεται δυναμική δέσμευση μνήμης για τον πίνακα `rotated_image[]` της περιστραμμένης εικόνας:
        <details>
        <summary></summary>

        ```c
        char *rotated_image = malloc(new_file_size * sizeof(char));

        if(rotated_image == NULL){
        fprintf(stderr, "Error while allocating memory");
        exit(1);
        }  
        ```

        </details>

#### Αρχικοποίηση *Headers* και *Other Data* στο `rotated_image[]`

1. Αφαιρώντας από το *new_file_size* το *new_image_size* βρίσκω το μέγεθος των *Headers* και *Other Data*. Αντιγράφω τα bytes που βρίσκονται σε αυτές τις θέσεις του `og_image[]` στο `rotated_image[]`: 
    <details>
    <summary></summary>

    ```c
    int o;
    for(o=0; o<(new_file_size - new_image_size); o++){
        rotated_image[o] = og_image[o];
    }
    ```

    </details>

2. Αλλάζω τα απαραίτητα στοιχεία στα *Ηeaders* με τον ίδιο τρόπο που αρχικοποίησα το *file_size* στο `og_image[]`: 
    <details>
    <summary> π.χ. </summary>

    ```c
    char *new_file_size_bytes;
    new_file_size_bytes = (char *) &new_file_size;
    rotated_image[2] = new_file_size_bytes[0];
    rotated_image[3] = new_file_size_bytes[1];
    rotated_image[4] = new_file_size_bytes[2];
    rotated_image[5] = new_file_size_bytes[3];
    ```

    </details>
    (είναι βέβαιο πως θα μπορούσα να φτιάξω μια συνάρτηση για την υλοποίηση της διαδικασίας της αλλαγής των στοιχείων, η αλήθεια είναι πως δεν ξέρω γιατί δεν το έκανα :see_no_evil: :hear_no_evil: :speak_no_evil:)

#### Δημιουργία του νέου *Pixel Array* στο `rotated_image[]`

1. <details>
    <summary>Μεταβλητές</summary>

    | **Τύπος** | **Όνομα**             | **Χρήση**                             | **Αρχική Τιμή**                    |
    |:---------:|:---------------------:|:-------------------------------------:|:----------------------------------:|
    | *int*     | go_to_pixel           | σταθερά                               | `width*3 - 3`                      |
    | *int*     | go_to_column          | μετριτής στήλης                       | `pixel2d_array_start + go_to_pixel`|
    | *int*     | height_counter        | μετριτής γραμμής                      | 0                                  |
    | *int*     | go_to_row             | σταθερά                               | `padding + width*3`                |
    | *int*     | k                     | μετριτής pixel bytes                  | 0                                  |
    | *int*     | m                     | μετριτής στοιχείου `rotated_image[]`  | `pixel2d_array_start`              |
    | *int*     | n                     | μετριτής padding                      | 0                                  |

    </details>

2. Η διαδικασία είναι απλή. Η τελευταία στήλη pixel του αρχικού πίνακα γίνεται πρώτη γραμμή του νέου, η προτελευταία στήλη δεύτερη γραμμή, και γενικότερα η *width - a* στήλη γίνεται η *a* γραμμή. Στο τέλος της γραμμής του νέου πίνακα προσθέτω πλήθος μηδενικών bytes ίσο με το *new_padding*. Για να υλοποιηθεί αυτή η διαδικασία στον 1D πίνακα που έχω δεσμεύσει θα χρειαστώ λίγη φαντασία για να το χειριστώ ως 2D:
    1. Αρχικά για να φτάσω στο πρώτο byte του πρώτου pixel στην «τελευταία στήλη» του *2D Pixel Array* του `og_image[]` πρέπει να προσθέσω στο *pixel2d_array_start* το γινόμενο του πλάτους επί το 3 (μέγεθος pixel σε bytes) μειωμένο κατά 3 (λόγω του ότι η διεύθυνση του στοιχείου που αναζητώ βρίσκεται 3 θέσεις αριστερότερα καθώς φτάνω χωρίς την αφαίρεση στην διεύθυνση του πρώτου byte του padding). Για να φτάσω στην αμέσως προηγούμενη «στήλη» μπορώ να αφαιρέσω ξανά από την ποσότητα αυτή το 3. Έτσι φτιάχνω έναν μετριτή που θα με βοηθήσει να βρίσκω τις «στήλες» του *2D Pixel Array* του `og_image[]` τον οποίο ονομάζω **go_to_column** και με κάθε επανάληψη κατά την αντιγραφή των στοιχείων μειώνεται κατά 3 ώσπου να γίνει ίσο με το *pixel2d_array_start* (δηλαδή να φτάσει να «δείχνει στην πρώτη στήλη»). Άρα χρησιμοποώ ένα `while` loop με συνθήκη `go_to_column >= pixel2d_array_start`. 

    2. Αφού φτάσω στην «στήλη» που επιθυμώ να αντιγράψω πρέπει να προσδιορίζω και σε ποιά «γραμμή» βρίσκομαι. Παρατηρώ ότι μπορώ να χρησιμοποήσω μια σταθερά που ισούται με `padding + width*3` και την ονομάζω **go_to_row**, η οποία ισούται με την απόσταση μεταξύ του πρώτου byte οποιουδήποτε pixel οποιασδήποτε στήλης με το πρώτο byte του pixel της αμέσως επόμενης γραμμής στην ίδια στήλη. Βοηθητικά ορίζω τον μετριτή γραμμής **height_counter** ο οποίος αυξάνει κατά 1 σε κάθε επανάληψη κατά την αντιγραφή αρχίζοντας από την τιμή 0 (πρώτη «γραμμή»).

    3. Μετά τον προσδιορισμό της «στήλης» και της «γραμμής» του pixel που αντιγράφω χρησιμοποιώ μια `for` loop ώστε να αντιγράψω και τα 3 bytes του κάθε pixel.
    
    4. Ο μετριτής στοιχείου του `rotated_image[]` **m** αυξάνει κατά 1 με την κάθε αντιγραφή (από `og_image[]`) ή προσθήκη (*padding*) οποιουδήποτε byte στο `rotated_image[]`.

    5. Στο τέλος της συμπλήρωσης της κάθε «γραμμής» προστίθεται το padding και συνεχίζεται η αντιγραφή της επόμενης «στήλης» του αρχικού *2D Pixel Array*.
    <details>
    <summary>Υλοποίηση διαδικασίας σε κώδικα</summary>

    ```c
    int go_to_pixel = width*3 - 3;
    int go_to_column = pixel2d_array_start + go_to_pixel;
    int height_counter = 0;
    int go_to_row = padding + width*3;
    int k, m, n;
    m = pixel2d_array_start;
    
    while(go_to_column >= pixel2d_array_start){
        for(height_counter=0; height_counter < height; height_counter++){
            for(k=0; k<3; k++){
                rotated_image[m] = og_image[ go_to_column + k + (height_counter * go_to_row)];
                m++;
            }
        }
        for(n=0; n < new_padding; n++){
                rotated_image[m] = 0;
                m++;
            }
    go_to_column -= 3;
    }
    ```

    </details>
