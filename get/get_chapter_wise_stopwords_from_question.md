CREATE FUNCTION get_chapter_wise_stopwords_from_question(topic_id INTEGER) 
    RETURNS JSON 
    LANGUAGE plpgsql
AS
$$ 
DECLARE 
    stopword_list TEXT[];
BEGIN
    RETURN (
        SELECT
            row_to_json(u) AS stopword_list 
        FROM
            (
                SELECT
                    ARRAY_AGG(word) AS stopwords,
                    COUNT(word) AS word_count 
                FROM
                    (
                        SELECT
                            *,
                            ((CAST(ndoc AS decimal) / (
                                SELECT
                                    COUNT(id) 
                                FROM
                                    "Question" 
                                WHERE
                                    "topicId" = topic_id 
                                    AND "deleted" = false)) * 100
                            )
                            ::FLOAT AS repr 
                        FROM
                            ts_stat(format('SELECT to_tsvector(''context_dict'', question) FROM "Question" WHERE "topicId" = %s', topic_id)) 
                        WHERE
                            LENGTH(word) > 3 
                        ORDER BY
                            repr DESC 
                    )
                    question_stat 
                WHERE
                    repr >= 7
            )
            u);
END;
$$;

ALTER FUNCTION get_chapter_wise_stopwords_from_question(integer) OWNER TO learner;
