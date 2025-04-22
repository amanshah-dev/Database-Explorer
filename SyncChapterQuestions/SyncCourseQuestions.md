create function "SyncCourseQuestions"(courseid integer) returns void
    language plpgsql
as
$$
DECLARE 
  rec RECORD;
BEGIN 
  FOR rec IN 
    SELECT "chapterId" from "SubjectChapter", "Subject" 
      where "SubjectChapter"."subjectId" = "Subject"."id"
        and "Subject"."courseId" = courseId
  LOOP
    PERFORM "SyncChapterQuestions"(rec."chapterId");
  END LOOP;
END
$$;

alter function "SyncCourseQuestions"(integer) owner to learner;
