create procedure update_retinaexpert(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_expert('Retina', achievementCategoryId, userIds);
END;
$$;

alter procedure update_retinaexpert(bigint, bigint[]) owner to learner;
