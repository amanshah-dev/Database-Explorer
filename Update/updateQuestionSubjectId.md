create function "updateQuestionSubjectId"() returns void
    language plpgsql
as
$$
BEGIN
  UPDATE "Question" 
  SET "subjectId" = "SubjectChapter"."subjectId"
  FROM "SubjectChapter"
  WHERE "SubjectChapter"."chapterId" = "Question"."topicId"
    AND "SubjectChapter"."subjectId" IN (53, 54, 55, 56)
    AND "Question"."subjectId" IS NULL;
END
$$;

alter function "updateQuestionSubjectId"() owner to learner;
