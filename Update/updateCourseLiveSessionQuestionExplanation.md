create function "updateCourseLiveSessionQuestionExplanation"(courseid integer) returns void
    language plpgsql
as
$$
DECLARE
  rec RECORD;
BEGIN
  FOR rec IN 
    SELECT "ChapterTest"."testId" from "ChapterTest", "SubjectChapter", "Subject" 
      where "SubjectChapter"."subjectId" = "Subject"."id"
        and "ChapterTest"."chapterId" = "SubjectChapter"."chapterId"
        and "Subject"."courseId" = courseId
  LOOP
    PERFORM "updateLiveSessionQuestionExplanation"(rec."testId");
  END LOOP;   
END
$$;

alter function "updateCourseLiveSessionQuestionExplanation"(integer) owner to learner;
