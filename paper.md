# A Database System to Search and Correlate Compromised Records Fast

[Gilbert Liu](https://github.com/gip0)

## Abstract

User data leaking incidents are quite common these days. While dozens of database from well known websites are publicly available, It is difficult to extract more information from these datasets due to heterogeneity of original data sources. A general database system that efficiently indexes and searches compromised records is described in this paper. Also a method to correlate records to show connectivity among a subset of such records is introduced. This research may benefit both security researchers and internet users in the perspective of privacy protection.

Keywords: Database, Indexing, Heterogeneity, Correlation

## 1 Introduction

Trojan horses were used from long ago to collect private information, such as credit card numbers, social security numbers and mothers' maiden names. These kind of attacks targeting on personal privacy information were mostly on a single-case basis, because of the difficulties in the process of gathering and extracting useful information from keylogging.

Hackers might not have ever dreamed such an era like nowadays. The problem of information gather and normalization solves itself in a blink of an eye, since the users and websites have already input sensitive information and stored in databases. All the hackers need to do is just to dump the databases. After 2009, more and more databases are compromised and available publicly, including a porn star HIV test database in 2011.[1] Partial lists of compromised databases can be found online.[2] [3] 

The massive database compromise leads to potential risk of identity theft. Since the records are publicly available, crime can proceed in silence, leaving the victims totally unaware until seeing their credit card bills. Most important thing of information compromise is that there is no take back. Once it spreads on the internet, it takes tremendous effort to remove. The best defense practice, from a personal perspective, is to maintain a proactive manner. Internet users need to know whether their information is compromised, and more importantly, what is compromised.

At the moment acquiring this information is not easy. By searching the most widely used email address "test@gmail.com", we find most of online leaked records searching services do not reveal any details.[4] [5] Simply telling the user "you are pwned" does not help at all. Although another service lists all the pastes containing the email address from pastebin.com, it is hard for unskilled user to find out what is leaked about himself. [6] Also, the user may have multiple email addresses or IDs on the internet. Not being able to correlate records with the only identifier from user input each time, and to return all the related records, will give users misleading information about their situation. On the other hand, security researchers need tools to organize these data from variety of sources for the very first step of research. It has been many years after trend of databases compromise started, but there are no current tools specifically for this purpose.

This is mainly because of the nature of dumped data is heterogeneity, which will be comprehensively discussed in following section. This determines such records may be problematic when importing to relational databases, or even to NoSQL ones.

This paper introduces an indexing method which allows efficiently searching and correlating such records. An experimental database system is also designed and implemented based on the characteristics of such leaked records for better illustration of the idea.

The research still has much to be done. For example unit tests and further benchmark for the database system, and the correlation part. This paper is a draft, and is to be updated over time.

## 2 Leaked Records

Samples of leaked records from variety of sources reveal some basic characteristics of such records that this research focus on. The China Software Developer Network (CSDN) records leaked in 2011 [7], the adobe user records in 2013 [8] are used. These two datasets are selected for many reasons. One is their fields are generic in some degree, which means, fields these datasets have likely exists in many other datasets, for example, username, password, and email address. Another reason is their formats represent the majority. Both of selected datasets are in plain text format and delimited with characters rarely appear in record content. Researching these generic properties will make the results most likely to hold for many other datasets.

### 2.1 Raw Data

As stated earlier, format heterogeneity is a most noticeable characteristic. Different datasets are likely to have different fields to each other. For example, as shown in Figure 2.1 and Figure 2.2 below, the CSDN records have email addresses and unencrypted passwords, while the Adobe records have user ID and password hints, but users' passwords were encrypted (can be reversed [9]).

![Figure 2.1](https://raw.githubusercontent.com/gip0/cutlassfish/master/res/fig_2_1.png)  
*Figure 2.1 Sample of Leaked Records from CSDN*

![Figure 2.2](https://raw.githubusercontent.com/gip0/cutlassfish/master/res/fig_2_2.png)  
*Figure 2.2 Sample of Leaked Records from Adobe*

Because of source heterogeneity, a global primary ID for each record does not come with the records. Also, duplicated and erroneous records, such as ill formed email addresses, among millions of them are hard to eliminate and could cause problems when analyzed.

Differences of fields make these records very troublesome to fit traditional relational database systems, although a table could be created with all possible fields and leaving those fields that a record does not have empty. Due to the same reason, a "collection" full of "documents" (mongodb terms) with inconsistent fields is not what we want. The implied information comes with original format when the records were dumped is another thing to study. Maintaining exact lines of records and not discarding any fields considered useless shall be one requirement of this database system.

## 3 Full Text Indexing and Searching

Full text indexing and searching technologies are overviewed in this section. Because of the inconvenience storing such records in traditional database systems, reviewing fulltext searching technologies is helpful to solve the problem.

There are two major categories of full text searching. The most commonly used one is brute force searching, also known as serial scanning. The other one has a much larger set of concepts, which is index and search.

Brute force searching is more of "online" processing. It does not store any information for future searches. Although searching rules and matching operations could be heavy sometimes, complicated regular expressions for example, they run all over again on each input. An example of brute force searching is the command line utility grep.

There are several basic concepts of index and search. In order to understand these concepts, consider a web search engine like Google. Each web page is a document, and words in document is called terms. The collection of documents is corpus. These concepts are illustrated with a sample matrix of documents versas terms in Figure 3.1 below.

![Figure 3.1](https://raw.githubusercontent.com/gip0/cutlassfish/master/res/fig_3_1.png)  
*Figure 3.1 sample matrix of documents versas terms*

For a large corpus, there will be a long list of all terms, and the matrix will be extremely sparse. Thus storing such a matrix is apparently inefficient.

Inverted index is an intuitive way to store terms versas documents. It has key of term id, and value of a list consists of document ids, of which has the term in it. Figure 3.2 shows the idea of inverted index. [10]

![Figure 3.2](https://raw.githubusercontent.com/gip0/cutlassfish/master/res/fig_3_2.png)  
*Figure 3.2 inverted index for example in Figure 3.1*

Advanced data structures can be used to improve the index. For example, to use a tree data structure to store all non-leaf nodes in the memory while all leaf nodes on disk, so that the expensive disk accesses can be minimized; or to encode terms with variable-length encodings, with which the most common terms are encoded into codes with shortest length, so that the index is compressed.

The searching process is straightforward too. First, the system needs to find the ID of the term user queries. Usually a hash function is used to calculate term ids from terms, so that the system does not have to look up in the database for term ID. Then, all document IDs that have this term is retrieved from the inverted index with the term ID from first step. Finally the system fetches documents with document IDs and return these documents to user to fulfill the request.

## 4 Solution

Clarifying some principles helps to limit the domain of the solution. Based on such principles, an experimental database system is designed to hold the records, indexes and possible future intermediate data. An indexing strategy is introduced and is compatible with the database system.

### 4.1 Principles

Regarding the problem of searching such records fast enough, some principles of solving the problem need to be stated. These principles are from the nature of the problem, but still not easy to realize.

Leaked records are mostly in plain text files which can be large. The Adobe leaked records file is more than 9GB. To search a substring through 9GB data or 150 millions lines in a sequential manner takes too long if read from hard drive. Searching in memory could be a much faster solution, but currently it is not affordable to hold more than one such file in the memory of a desktop computer. Therefore a database system is needed, and it needs to be well crafted to search multiple gigabytes of data in milliseconds.

In most cases, the leaked records from each dataset will be imported only once, read a lot of times, and not deleted forever. So the database system is designed to focus on performance of randomly selecting records, but not inserting.

Also data format is an immediate problem. Datasets of leaked records are mostly in plain text format, others are dump files of different databases systems. Plain text format is a good choice to store records because of the nature of such datasets. Using plain text and without modification on lines of records can maximize original information preserved.

Last, exact matching instead of other matching methods, e.g. substring matching, wildcards or regular expression. Searching such records is similar to searching a journal database with very vague keywords. User of the system is supposed only to search for records related to himself. So that queries matching shall always be exact.

### 4.2 Storing

The storage system is text files based, because the raw records are mostly in plain text files. In order to keep original structural information, each record is saved in database with an record ID and no further modification.

Database is an important term in the context of this system. Database refers to an collection of index and data files, with some meta data. Each database has its own set of unique record IDs, which are 64 bits long and represented in 8-byte strings. Each database also have one index, a file consists of multiple pairs of an 8-byte block ID and an 8-byte record ID. In databases, data is stored in small files called blocks. The default size of block is 64kB. In each block, records are prepended with record IDs, and a newline character at the end, also known as "\n".

![Figure 4.1](https://raw.githubusercontent.com/gip0/cutlassfish/master/res/fig_4_1.png)  
*Figure 4.1 How Records Is Saved in Block and Databases*

8-byte record ID is necessary. 4 bytes, or 32 bits, can represent 0 to 4.2 billions. In the 9GB file from Adobe in 2013, there are 155 millions records. If they are all imported into a such database, about 3.6% of 32 bits record IDs will be used. Therefore 32 bits record ID is not a good idea. More and more records will leak in coming few years, maybe 8 bytes wouldn¡¯t be enough in the future.

Block size is a key parameter of the storage system that influences overall performance. Block size shall be set to integer multiples of file system block size to avoid space waste. The database system uses block as an atomic unit of disk I/O. It takes advantages of file system when locating a piece of data by using relatively smaller block files. Smaller block (file) means more precisely locating of data, and shorter disk read time. But too many small files may downgrade the file system performance after all. Properly set the block size of the system and of the file system can help decreasing average record access time. Benchmark results are discussed in section 5 with comparison on different block sizes.

The storage system has two modes for each database: sequential and ordered. Sequential means the system supposes inserted record has an incremental record ID. User needs to assign record ID for each record, and to ensure the IDs are incremental. This mode is used when inserting leaked records to the system. A database in ordered mode will automatically keep records ordered by primary key over all blocks, no matter what primary key is assigned to record by user. This mode is used to store terms v.s. records index, which is discussed with details in 4.3.

For quicker fetching records, each database has an index of block ID v.s. record ID. Such indexes are loaded into memory when a database connection is created. A database index is different from an index database, which is a database in ordered mode storing terms v.s. records index specifically.

### 4.3 Indexing

Searching leaked records have fundamental difference comparing to searching natural language materials. Not only because of exact matching, but also that only very limited number of terms can be extracted from each record. Existing index and search methods are obviously not designed for such half structured datasets.

Indexing is to create connections between keywords and records by giving each term an ID. Uniqueness is important for term ID. One possible way is to assign an unique ID to each term manually. But this is hardly feasible with the pursuit of performance. Since such ID and term pairs can only be saved on disks, multiple disk accesses just for retrieving term IDs are not acceptable.

As cited in previous section, hash functions can be used to calculate term IDs. Hash functions have collision and speed problem when massively used. Collision means that sending two different inputs and getting a same hash from the hash function. This is bad in every aspects when using a hash function with high collision rate in term ID calculation. Collisions are inevitable simply because all hash functions have much more possible inputs than possible outputs. Speed also becomes a concern when a hash function is intensively used.

Collision is not necessarily a problem since this is not safety-critical application. If collision happens with small enough possibility, it is acceptable. SMHasher tests are a series of tests designed for evaluating hash functions from performance, collision and distribution perspective. [11] And quantitative scores are given to hash functions. [12] Hence xxhash64 is chosen for calculating 64 bits term IDs.

![Figure 4.2](https://raw.githubusercontent.com/gip0/cutlassfish/master/res/fig_4_2.png)  
*Figure 4.2 Indexing Records with Term IDs and Record IDs*

In terms of key-value pair, the key of index is term ID. The record ID of which has the term is the value. Multiple records having a same term is possible, so there will be more than one such pairs with same key. To fit in the framework of the storage system, index uses a different database in ordered mode than the leaked records, see Figure 4.2. This is because indexes are not inserted with primary keys ordered.

## 5 Benchmark

*This section contains partial benchmark results and some are not thoroughly discussed at the moment. Updates will be applied over time.*

The database system has two important parameters: mode and block size. Benchmark of record I/O time against these two parameters were performed. Each parameter set runs ten loops. In each loop, ten thousand records are inserted to a database with timing, then followed by ninety thousand other records insertion. At this point, ten thousand selections with random record ID are timed, and then the next loop continues. The record IDs for selection are chosen randomly from the ninety thousand records inserted in the same loop in ordered mode, but from zero to last inserted record ID in sequential mode.

![Figure 5.1](https://raw.githubusercontent.com/gip0/cutlassfish/master/res/fig_5_1.png)  
*Figure 5.1 10k Records Insertion Time v.s. Existing Records (Sequential Mode)*

Figure 5.1 above shows insertion timing result of a database in sequential mode with different block sizes. The smallest block size, 4KB, does not perform well because of frequent disk I/O.

![Figure 5.2](https://raw.githubusercontent.com/gip0/cutlassfish/master/res/fig_5_2.png)  
*Figure 5.2 10k Records Selection Time v.s. Existing Records (Sequential Mode)*

With block size of 256KB, every disk access is more expensive than lower block size. Since test data is only one million, and there are not too many blocks of 256KB, these selection are not distributed evenly into every block. This is why the result of 256KB block size is not even like others with smaller block sizes.

![Figure 5.3](https://raw.githubusercontent.com/gip0/cutlassfish/master/res/fig_5_3.png)  
*Figure 5.3 10k Records Insertion Time v.s. Existing Records (Ordered Mode)*

In ordered mode, shown in Figure 5.3, the overall trend of insertion time is growing. This is because in order to keep records ordered blocks need to be split into two sometimes when inserting new records. The database has more blocks, the performance on this is poorer, from the comparison between the 4KB and 16KB. The instability of 256KB is because the total number of blocks is not large enough, just like the selection time in sequential mode above.

![Figure 5.4](https://raw.githubusercontent.com/gip0/cutlassfish/master/res/fig_5_4.png)  
*Figure 5.4 10k Records Selection Timve v.s. Existing Records (Ordered Time)*

From Figure 5.4, the selection time of 256KB and 64KB-block databases are stable. But with 16KB and 4KB blocks, the time of selection is growing, although very slowly.

Benchmark results in this section are useful if user needs to tune the performance of the database system. Generally, larger blocks take more time on selecting records, but smaller blocks do not perform well on inserting. Constraints from file systems shall also be considered. Since the database system is reading-heavy, a smaller block size is recommended.

## 6 Future Directions

There are some possible future working directions on this research as listed below. Due to some reasons, this research is currently pending.

The importance of finding relativity of records is clarified in previous sections. To calculate relativity between two records, the idea of tf-idf can be borrowed. Tf is short for term frequency. Basic assumption of term frequency is that the more a same term occurs in two documents, the more related they are. While the idf, inverse document frequency, is the rareness of a term across all over documents. [13] Detailed theory and algorithm implementation that fits the database system can be an interesting direction.

The front-end of this database system, such as a web interface, is another direction. The user interaction process needs to be carefully designed so that user will not be able to abuse the web service to download other¡¯s private information.

## 7 Conclusion

This paper introduces a database system to efficiently search and correlate leaked records. Samples of publicly available leaked databases were shown, and fundamental properties of such databases were concluded. This was followed by a brief introduction of full text indexing and searching technologies. The problem to solve was finely defined, then the database system was described with details. Basic benchmark results were analyzed, some were left out temporarily. Two possible future research directions were mentioned, therefore the database system can either be used for research purposes or be the back-end of a public web system that educates internet users about privacy protection.

To conclude, the database system introduced is an experimental program aiming at performance of searching leaked internet records, but the indexing and searching method can be used in future research.

## Reference

[1] Chen, A. (11, March 30). Porn Star HIV Test Database Leaked. Retrieved from [http://gawker.com/5787392/porn-star-hiv-test-database-leaked](http://gawker.com/5787392/porn-star-hiv-test-database-leaked)  
[2] Have I been pwned? Pwned websites. (n.d.). Retrieved from [https://haveibeenpwned.com/PwnedWebsites](https://haveibeenpwned.com/PwnedWebsites)  
[3] The Password Project. (n.d.). leaked_password_lists_and_dictionaries - The Password Project. Retrieved October 1, 2014, from [http://thepasswordproject.com/leaked_password_lists_and_dictionaries](http://thepasswordproject.com/leaked_password_lists_and_dictionaries)  
[4] LastPass. (n.d.). LastPass - Adobe Email Checker. Retrieved October 1, 2014, from [https://lastpass.com/adobe/](https://lastpass.com/adobe/)  
[5] PwnedList.com - Have Your Accounts Been Compromised? (n.d.). Retrieved from [https://pwnedlist.com/query](https://pwnedlist.com/query)  
[6] Have I been pwned? Check if your email has been compromised in a data breach. (n.d.). Retrieved October 1, 2014, from [https://haveibeenpwned.com](https://haveibeenpwned.com)  
[7] Wikipedia. (n.d.). CSDN - Wikipedia, the free encyclopedia. Retrieved October 1, 2014, from [http://en.wikipedia.org/wiki/CSDN#User_information_leakage](http://en.wikipedia.org/wiki/CSDN#User_information_leakage)  
[8] Wikipedia. (n.d.). Adobe Systems - Wikipedia, the free encyclopedia. Retrieved October 1, 2014, from [http://en.wikipedia.org/wiki/Adobe_Systems#Source_code_and_customer_data_breach](http://en.wikipedia.org/wiki/Adobe_Systems#Source_code_and_customer_data_breach)  
[9] Ducklin, P. (2013, November 4). Anatomy of a password disaster ¨C Adobe¡¯s giant-sized cryptographic blunder | Naked Security. Retrieved from [http://nakedsecurity.sophos.com/2013/11/04/anatomy-of-a-password-disaster-adobes-giant-sized-cryptographic-blunder/](http://nakedsecurity.sophos.com/2013/11/04/anatomy-of-a-password-disaster-adobes-giant-sized-cryptographic-blunder/)  
[10] Das, A., & Jain, A. (n.d.). Indexing The World Wide Web: The Journey So Far. Retrieved from [http://static.googleusercontent.com/media/research.google.com/en/us/pubs/archive/37043.pdf](http://static.googleusercontent.com/media/research.google.com/en/us/pubs/archive/37043.pdf)  
[11] The Linux Documentation Project. (n.d.). Filesystems. Retrieved from [http://www.tldp.org/LDP/sag/html/filesystems.html](http://www.tldp.org/LDP/sag/html/filesystems.html)  
[12] SMHasher. (n.d.). Retrieved October 1, 2014, from [https://code.google.com/p/smhasher/wiki/SMHasher](from https://code.google.com/p/smhasher/wiki/SMHasher)  
[13] xxhash. (n.d.). Retrieved October 1, 2014, from [https://code.google.com/p/xxhash/](https://code.google.com/p/xxhash/)  
[14] tf¨Cidf - Wikipedia, the free encyclopedia. (n.d.). Retrieved October 13, 2014, from [http://en.wikipedia.org/wiki/Tf%E2%80%93idf](http://en.wikipedia.org/wiki/Tf%E2%80%93idf)  
