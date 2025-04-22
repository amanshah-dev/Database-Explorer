create function removetestquestionws(testid integer) returns void
    language plpgsql
as
$$
BEGIN
  update "Question" set "question" = regexp_replace("question", '<br />\r\n<br />\r\n<br />\r\n<br />\r\n&nbsp;$', ''), "updatedAt" = current_timestamp 
  where "question" ~ '<br />\r\n<br />\r\n<br />\r\n<br />\r\n&nbsp;$' and "id" in (select "questionId" from "TestQuestion" where "testId" in (testId));

  update "Question" set "question" = regexp_replace("question", '<br />\r\n<br />\r\n<br />\r\n&nbsp;$', ''), "updatedAt" = current_timestamp 
  where "question" ~ '<br />\r\n<br />\r\n<br />\r\n&nbsp;$' and "id" in (select "questionId" from "TestQuestion" where "testId" in (testId));

  update "Question" set "question" = regexp_replace("question", '<br />\r\n<br />\r\n&nbsp;$', ''), "updatedAt" = current_timestamp 
  where "question" ~ '<br />\r\n<br />\r\n&nbsp;$' and "id" in (select "questionId" from "TestQuestion" where "testId" in (testId));

  update "Question" set "question" = regexp_replace("question", '<br />\r\n&nbsp;$', ''), "updatedAt" = current_timestamp 
  where "question" ~ '<br />\r\n&nbsp;$' and "id" in (select "questionId" from "TestQuestion" where "testId" in (testId));

  update "Question" set "question" = regexp_replace("question", '<p><br />\r\n<br />\r\n<br />\r\n<br />\r\n&nbsp;</p>$', ''), "updatedAt" = current_timestamp 
  where "question" ~ '<p><br />\r\n<br />\r\n<br />\r\n<br />\r\n&nbsp;</p>$' and "id" in (select "questionId" from "TestQuestion" where "testId" in (testId));

  update "Question" set "question" = regexp_replace("question", '<p><br />\r\n<br />\r\n<br />\r\n&nbsp;</p>$', ''), "updatedAt" = current_timestamp 
  where "question" ~ '<p><br />\r\n<br />\r\n<br />\r\n&nbsp;</p>$' and "id" in (select "questionId" from "TestQuestion" where "testId" in (testId));

  update "Question" set "question" = regexp_replace("question", '<p><br />\r\n<br />\r\n&nbsp;</p>$', ''), "updatedAt" = current_timestamp 
  where "question" ~ '<p><br />\r\n<br />\r\n&nbsp;</p>$' and "id" in (select "questionId" from "TestQuestion" where "testId" in (testId));

  update "Question" set "question" = regexp_replace("question", '<p><br />\r\n&nbsp;</p>$', ''), "updatedAt" = current_timestamp 
  where "question" ~ '<p><br />\r\n&nbsp;</p>$' and "id" in (select "questionId" from "TestQuestion" where "testId" in (testId));

  update "Question" set "question" = regexp_replace("question", '<br />\r\n&nbsp;$', ''), "updatedAt" = current_timestamp 
  where "question" ~ '<br />\r\n&nbsp;$' and "id" in (select "questionId" from "TestQuestion" where "testId" in (testId));

  update "Question" set "question" = regexp_replace("question", '<p><br />\r\n<br />\r\n<br />\r\n<br />\r\n&nbsp;</p>\r\n$', ''), "updatedAt" = current_timestamp 
  where "question" ~ '<p><br />\r\n<br />\r\n<br />\r\n<br />\r\n&nbsp;</p>\r\n$' and "id" in (select "questionId" from "TestQuestion" where "testId" in (testId));

  update "Question" set "question" = regexp_replace("question", '<p><br />\r\n<br />\r\n<br />\r\n&nbsp;</p>\r\n$', ''), "updatedAt" = current_timestamp 
  where "question" ~ '<p><br />\r\n<br />\r\n<br />\r\n&nbsp;</p>\r\n$' and "id" in (select "questionId" from "TestQuestion" where "testId" in (testId));

  update "Question" set "question" = regexp_replace("question", '<p><br />\r\n<br />\r\n&nbsp;</p>\r\n$', ''), "updatedAt" = current_timestamp 
  where "question" ~ '<p><br />\r\n<br />\r\n&nbsp;</p>\r\n$' and "id" in (select "questionId" from "TestQuestion" where "testId" in (testId));

  update "Question" set "question" = regexp_replace("question", '<p><br />\r\n&nbsp;</p>\r\n$', ''), "updatedAt" = current_timestamp 
  where "question" ~ '<p><br />\r\n&nbsp;</p>\r\n$' and "id" in (select "questionId" from "TestQuestion" where "testId" in (testId));
END;
$$;

alter function removetestquestionws(integer) owner to learner;
